# 58386 sc low rewards claimed during deallocation remain stranded on strategy and unaccounted

**Submitted on Nov 1st 2025 at 20:38:30 UTC by @zcai for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58386
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The strategy claims reward tokens directly to its own address during deallocation operations but lacks a mechanism to forward these already-claimed rewards to the vault. This creates a persistent desynchronization where external value accrues to the strategy but remains untracked in vault accounting, effectively stranding earned rewards that belong to users.

## Vulnerability Details

During deallocation operations, the `_deallocate()` function calls `rewarder.withdraw()` with the claim parameter set to true, which transfers any pending reward tokens directly to the strategy contract address. However, the reward synchronization mechanism in `_claimRewards()` only handles pending rewards that remain at the rewarder contract level.

The issue manifests through the following sequence:

1. When `_deallocate()` executes, it calls `rewarder.withdraw(address(this), sharesNeeded, true)` on line 76
2. The `claim=true` parameter causes any pending reward tokens to be transferred to the strategy contract
3. Subsequently, `_claimRewards()` uses `rewarder.getReward(address(this), address(MYT), false)` which only forwards pending rewards from the rewarder to the vault
4. Since rewards were already claimed during deallocation, no pending rewards remain at the rewarder level
5. The reward tokens transferred to the strategy during deallocation remain stranded with no mechanism to forward them

The `realAssets()` function excludes reward token balances entirely, reporting only the underlying asset value and ignoring any reward tokens held by the strategy. This creates a permanent accounting discrepancy where the strategy holds value that is never reflected in vault calculations or distributed to users.

## Impact Details

Users experience passive value loss as reward tokens earned by their deposited capital become permanently inaccessible. These stranded rewards accumulate over time on the strategy contract but are excluded from the vault's asset calculations and cannot be claimed through the existing reward distribution mechanism. While principal assets remain unaffected, the loss of reward value represents a direct economic impact to vault participants.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L76>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L95-98>

## Link to Proof of Concept

<https://gist.github.com/i-am-zcai/6b24ee4133605f72948ab2c7fb9677ef>

## Proof of Concept

## Proof of Concept

`src/test/poc/TokeAutoEth_StaleRewards.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {TokeAutoEthStrategy} from "../../strategies/mainnet/TokeAutoEth.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {IERC4626} from "../../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {TestERC20} from "../mocks/TestERC20.sol";

/// Mock ERC4626 vault representing autoETH; also acts as ERC20 share token
contract MockAutoEthVault is ERC20, IERC4626 {
    address public immutable _asset;

    constructor(address asset_) ERC20("MockAutoETH", "mAUTOETH") {
        _asset = asset_;
    }

    // IERC4626 minimal implementation
    function asset() external view override returns (address) { return _asset; }
    function totalAssets() public view override returns (uint256) { return IERC20(_asset).balanceOf(address(this)); }
    function convertToShares(uint256 assets) public pure override returns (uint256) { return assets; }
    function convertToAssets(uint256 shares) public pure override returns (uint256) { return shares; }
    function maxDeposit(address) external pure override returns (uint256) { return type(uint256).max; }
    function previewDeposit(uint256 assets) external pure override returns (uint256) { return assets; }
    function deposit(uint256 assets, address receiver) external override returns (uint256 shares) {
        // Assumes assets already transferred to this contract by the router.
        shares = assets;
        _mint(receiver, shares);
    }
    function maxMint(address) external pure override returns (uint256) { return type(uint256).max; }
    function previewMint(uint256 shares) external pure override returns (uint256) { return shares; }
    function mint(uint256 shares, address receiver) external override returns (uint256 assets) {
        assets = shares;
        require(IERC20(_asset).transferFrom(msg.sender, address(this), assets), "pull fail");
        _mint(receiver, shares);
    }
    function maxWithdraw(address owner) external view override returns (uint256) { return convertToAssets(balanceOf(owner)); }
    function previewWithdraw(uint256 assets) external pure override returns (uint256) { return assets; }
    function withdraw(uint256 assets, address receiver, address) external override returns (uint256 shares) {
        shares = assets;
        require(totalAssets() >= assets, "insufficient vault assets");
        IERC20(_asset).transfer(receiver, assets);
    }
    function maxRedeem(address owner) external view override returns (uint256) { return balanceOf(owner); }
    function previewRedeem(uint256 shares) external pure override returns (uint256) { return shares; }
    function redeem(uint256 shares, address receiver, address) external override returns (uint256 assets) {
        assets = shares;
        require(totalAssets() >= assets, "insufficient vault assets");
        IERC20(_asset).transfer(receiver, assets);
    }
}

/// Mock Autopilot Router implementing the subset used by the strategy
contract MockAutopilotRouter {
    function depositMax(IERC4626 vault, address to, uint256) external returns (uint256 sharesOut) {
        address asset_ = vault.asset();
        uint256 bal = IERC20(asset_).balanceOf(msg.sender);
        // Pull all available assets from caller (strategy) into the vault
        require(IERC20(asset_).transferFrom(msg.sender, address(vault), bal), "router pull fail");
        // Call deposit on the vault to mint shares to `to`
        sharesOut = vault.deposit(bal, to);
    }

    function stakeVaultToken(IERC4626, uint256) external pure returns (uint256) { return 0; }
    function withdrawVaultToken(IERC4626, IMainRewarder, uint256, bool) external pure returns (uint256) { return 0; }
}

/// Minimal subset of extra interfaces from Tokemak
interface IMainRewarder {
    function stake(address account, uint256 amount) external;
    function withdraw(address account, uint256 amount, bool claim) external;
    function getReward(address account, address recipient, bool claimExtras) external;
    function earned(address account) external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
}

/// Mock rewarder paying rewards in a fixed ERC20 token
contract MockRewarder is IMainRewarder {
    IERC20 public immutable rewardToken;
    mapping(address => uint256) public pending;
    mapping(address => uint256) public staked;

    constructor(IERC20 _rewardToken) { rewardToken = _rewardToken; }

    function stake(address account, uint256 amount) external override { staked[account] += amount; }

    function withdraw(address account, uint256 amount, bool claim) external override {
        require(staked[account] >= amount, "not enough staked");
        staked[account] -= amount;
        if (claim) {
            uint256 p = pending[account];
            if (p > 0) {
                pending[account] = 0;
                require(rewardToken.transfer(account, p), "reward xfer");
            }
        }
    }

    function getReward(address account, address recipient, bool) external override {
        uint256 p = pending[account];
        if (p > 0) {
            pending[account] = 0;
            require(rewardToken.transfer(recipient, p), "reward xfer");
        }
    }

    function earned(address account) external view override returns (uint256) { return pending[account]; }
    function balanceOf(address account) external view override returns (uint256) { return staked[account]; }

    // test helper
    function setPending(address account, uint256 amount) external { pending[account] = amount; }
}

/// Oracle stub for constructor type compatibility
contract MockOracle { function getPriceInEth(address) external pure returns (uint256 price) { return 1e18; } }

contract TokeAutoEth_StaleRewards_POC is Test {
    TestERC20 internal weth;         // underlying asset
    TestERC20 internal rewardToken;  // rewards token
    MockAutoEthVault internal autoEth;
    MockAutopilotRouter internal router;
    MockRewarder internal rewarder;
    MockOracle internal oracle;

    TokeAutoEthStrategy internal strat;

    address internal vault = address(0xBEEF); // pretend MYT vault address (EOA is fine)

    function setUp() public {
        // Deploy tokens and mocks
        weth = new TestERC20(0, 18);
        rewardToken = new TestERC20(0, 18);
        autoEth = new MockAutoEthVault(address(weth));
        router = new MockAutopilotRouter();
        rewarder = new MockRewarder(IERC20(address(rewardToken)));
        oracle = new MockOracle();

        // Strategy params
        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoEth",
            protocol: "Tokemak",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0 // no slippage to simplify full deallocation
        });

        // Permit2 can be any nonzero address for constructor guard
        address permit2 = address(0x1234);

        strat = new TokeAutoEthStrategy(
            vault,
            params,
            address(autoEth),
            address(router),
            address(rewarder),
            address(weth),
            address(oracle),
            permit2
        );
    }

    function test_poc_stale_rewards_TokeAutoEth() public {
        uint256 depositAmount = 100e18;

        // Seed strategy with WETH to allocate
        deal(address(weth), address(strat), depositAmount);

        // Vault calls allocate on the strategy
        vm.startPrank(vault);
        bytes memory prev = abi.encode(0);
        (bytes32[] memory ids, int256 change) = strat.allocate(prev, depositAmount, bytes4(0), vault);
        vm.stopPrank();
        assertEq(ids.length, 1, "ids");
        assertEq(uint256(change), depositAmount, "alloc change");
        // All shares should be staked in rewarder
        assertEq(rewarder.balanceOf(address(strat)), depositAmount, "staked");
        // Vault now holds the WETH deposited into the ERC4626
        assertEq(IERC20(address(weth)).balanceOf(address(autoEth)), depositAmount, "vault weth bal");

        // Accrue some pending rewards at the rewarder for the strategy
        // Fund the rewarder to be able to pay them out
        deal(address(rewardToken), address(rewarder), 50e18);
        rewarder.setPending(address(strat), 50e18);
        assertEq(rewarder.earned(address(strat)), 50e18, "pending before");

        // Deallocate fully: this will call rewarder.withdraw(..., claim=true) which transfers rewards to the strategy
        vm.startPrank(vault);
        prev = abi.encode(depositAmount);
        (ids, change) = strat.deallocate(prev, depositAmount, bytes4(0), vault);
        vm.stopPrank();
        assertEq(uint256(-change), depositAmount, "dealloc change");

        // Rewards have been sent to the strategy itself during deallocation
        uint256 stranded = IERC20(address(rewardToken)).balanceOf(address(strat));
        assertEq(stranded, 50e18, "rewards should be on strategy");
        assertEq(IERC20(address(rewardToken)).balanceOf(vault), 0, "vault should not have received rewards");
        assertEq(rewarder.earned(address(strat)), 0, "no more pending at rewarder");

        // Now try the provided reward sync path. It cannot move already-held rewards (earned==0)
        strat.claimRewards();
        assertEq(IERC20(address(rewardToken)).balanceOf(address(strat)), stranded, "rewards remain stranded on strategy");
        assertEq(IERC20(address(rewardToken)).balanceOf(vault), 0, "vault still has no rewards");

        // Additionally, realAssets() excludes reward token balances entirely
        uint256 real = strat.realAssets();
        assertEq(real, 0, "realAssets excludes stranded rewards");
    }
}
```
