# 56983 sc low tokemak rewards sent to myt vault contract not strategy rewards stranded

**Submitted on Oct 22nd 2025 at 12:05:43 UTC by @yesofcourse for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56983
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

`TokeAutoEthStrategy._claimRewards()` claims Tokemak incentives to the **MYT vault contract address** instead of to the **strategy** itself.

Because the vault doesn’t handle arbitrary reward tokens, these tokens are never converted to underlying nor credited to depositors, causing a silent APR drag.

In production this leads to **permanent freezing of unclaimed yield** at the vault address unless an out-of-band sweep is performed.

## Vulnerability Details

The strategy routes rewards to `address(MYT)` (the ERC-4626/Morpho vault) rather than to the strategy:

```solidity
// src/strategies/mainnet/TokeAutoEth.sol
function _claimRewards() internal override returns (uint256 rewardsClaimed) {
    rewardsClaimed = rewarder.earned(address(this));
    rewarder.getReward(address(this), address(MYT), false); // recipient = MYT vault (bug)
}
```

* In this architecture, **strategies** are responsible for realizing third-party incentives: claim them **to the strategy**, swap to the vault’s **underlying** (WETH), and return value to the vault (or re-deposit) so share price reflects rewards.
* By sending rewards directly to the **vault contract** (a component that only accounts in underlying), the rewards arrive as an **untracked arbitrary ERC-20**. There’s no logic in the shown codebase to sweep/convert such tokens at the vault; consequently, the value is not realized or credited.

**The PoC below simulates:** `TokeAutoEthStrategy_RewardMisroute_PoC` funds a mock rewarder, sets `earned[strategy] = R`, calls `claimRewardsPublic()` (which exposes `_claimRewards()`), and verifies:

* strategy reward-token balance **does not** increase,
* the “vault” address receives **exactly `R`** reward tokens,
* `earned[strategy]` is zeroed.

This directly demonstrates the misroute and confirms that claiming rewards **does not** benefit depositors.

## Impact Details

**Impact category:** **Permanent freezing of unclaimed yield.**

* Incentive tokens accumulate at the vault address and **are not realized into underlying**, so depositors’ share price/APR **does not** reflect rewards.
* Over time, the USD value of stranded rewards can become material (∑ rewards × token price), and compounding benefits are also lost.

## References

* Vulnerable function: `src/strategies/mainnet/TokeAutoEth.sol` -> `_claimRewards()` <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L95-L98>

  ```solidity
  rewarder.getReward(address(this), address(MYT), false);
  ```

## Proof of Concept

## Proof of Concept

In `src/test/strategies/TokeAutoEthStrategy.t.sol`, add the transfer() function to the IERC20 interface at the beginning:

```solidity
interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
}
```

Then paste the following at the end of that file:

```solidity

// ---------- PoC: misrouted Tokemak rewards go to MYT vault (not the strategy) ----------
import {Test} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// --- minimal local mocks ---
contract MockERC20 is ERC20 {
    constructor(string memory n, string memory s) ERC20(n, s) {}
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

// minimal mock rewarder used by the PoC
contract MockRewarder {
    IERC20 public immutable REWARD;

    // Public mapping already exposes: function earned(address) external view returns (uint256)
    mapping(address => uint256) public earned;

    constructor(address reward) { REWARD = IERC20(reward); }

    function setEarned(address acct, uint256 amt) external { earned[acct] = amt; }

    function getReward(address acct, address to, bool) external returns (uint256 amt) {
        amt = earned[acct];
        if (amt == 0) return 0;
        earned[acct] = 0;
        require(REWARD.transfer(to, amt), "transfer failed");
    }

    function rewardToken() external view returns (address) { return address(REWARD); }
}


// Expose _claimRewards() for testing (no other behavior changed)
contract TokeAutoEthStrategyHarness is TokeAutoEthStrategy {
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

    function claimRewardsPublic() external returns (uint256) {
        return _claimRewards(); // internal in base, exposed here
    }
}

contract TokeAutoEthStrategy_RewardMisroute_PoC is Test {
    // Local “vault” address to show rewards going to the wrong place
    address public constant MYT_VAULT = address(0xA11CE);

    MockERC20 rewardToken;      // fake TOKE (or any reward token)
    MockERC20 autoEth;          // dummy AUTO-ETH token to satisfy constructor approvals
    MockERC20 weth;             // dummy WETH to satisfy constructor approvals
    MockRewarder rewarder;
    TokeAutoEthStrategyHarness strat;

    function setUp() public {
        // deploy mocks
        rewardToken = new MockERC20("Reward", "RWD");
        autoEth     = new MockERC20("autoETH", "AETH");
        weth        = new MockERC20("WETH", "WETH");
        rewarder    = new MockRewarder(address(rewardToken));

        // strategy params (values mostly irrelevant for this PoC)
        IMYTStrategy.StrategyParams memory p = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoEth",
            protocol: "tokemak",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 1
        });

        // Router/oracle/permit2 can be any addresses for this PoC (unused by _claimRewards)
        strat = new TokeAutoEthStrategyHarness(
            MYT_VAULT,
            p,
            address(autoEth),
            address(0xBEEFBEEF),    // router (not used in this test)
            address(rewarder),      // <-- our mock rewarder
            address(weth),
            address(0xF00D),        // oracle (not used in this test)
            address(0xDEAD)         // permit2 (not used in this test)
        );
    }

    /// Proves that _claimRewards sends tokens to the MYT vault, not the strategy.
    function test_rewards_are_sent_to_vault_not_strategy() public {
        uint256 R = 100e18;

        // fund the rewarder and declare the strategy has earned R
        rewardToken.mint(address(rewarder), R);
        rewarder.setEarned(address(strat), R);

        uint256 stratBefore = rewardToken.balanceOf(address(strat));
        uint256 vaultBefore = rewardToken.balanceOf(MYT_VAULT);

        // act: claim rewards (calls rewarder.getReward(..., address(MYT), ...))
        strat.claimRewardsPublic();

        // assert: strategy did NOT receive rewards
        assertEq(rewardToken.balanceOf(address(strat)), stratBefore, "strategy unexpectedly received rewards");

        // assert: the MYT vault did receive the rewards (misrouted)
        assertEq(rewardToken.balanceOf(MYT_VAULT), vaultBefore + R, "rewards not sent to vault as in bug");

        // (optional) show that earned is zeroed
        assertEq(rewarder.earned(address(strat)), 0, "earned not cleared");
    }
}
```

Run with `forge test --match-contract TokeAutoEthStrategy_RewardMisroute_PoC -vv`:

```
Ran 1 test for src/test/strategies/TokeAutoETHStrategy.t.sol:TokeAutoEthStrategy_RewardMisroute_PoC
[PASS] test_rewards_are_sent_to_vault_not_strategy() (gas: 105401)
```
