# 58089 sc low arithmetic underflow revert in deallocate&#x20;

**Submitted on Oct 30th 2025 at 15:18:24 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58089
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The function `_deallocate` can revert due to an underflow.

The following code may revert:

```solidity
uint256 shareDiff = actualSharesHeld - sharesNeeded;
```

This is possible when `sharesNeeded > actualSharesHeld`.

## Vulnerability Details

Code snippet:

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

The issue is that when the function is trying to calculate the share difference, the code does not expect with `sharesNeeded` being of higher value than `actualSharesHeld`. There is no guard or validation before the subtraction to prevent this underflow.

The function should check or clamp the amount that is attempted to be withdrawn before performing the substraction.

## Impact Details

The impact is Denial of Service as deallocations will cause to fail unexpectedly.

Freezing funds for at least 24 hours, until the strategy holds enough shares.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L70>

## Proof of Concept

## Proof of Concept

Add this PoC in path `src/test/strategies/PoC.t.sol`.

This PoC demonstrates how deallocation may revert with 0x11 panic revert.

Run with `forge test --match-test test_deallocate_underflow_reverts -vvvv`

PoC:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";

/// ---------------------------
/// Minimal interfaces & utils
/// ---------------------------
interface IERC20Minimal {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function mint(address to, uint256 amount) external;
}

/// ---------------------------
/// Minimal ERC20 for mocks
/// ---------------------------
contract MockERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "balance");
        if (from != msg.sender) {
            require(allowance[from][msg.sender] >= amount, "allowance");
            allowance[from][msg.sender] -= amount;
        }
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

/// ---------------------------
/// Minimal vault (MYT) stub
/// ---------------------------
interface IVaultV2 {
    function asset() external view returns (address);
}

contract DummyVault is IVaultV2 {
    address public override asset;
    constructor(address _asset) {
        asset = _asset;
    }
}

/// ---------------------------
/// Interfaces used by strategy
/// ---------------------------
interface IMainRewarder {
    function balanceOf(address a) external view returns (uint256);
    function withdraw(address a, uint256 shares, bool claim) external;
    function stake(address a, uint256 shares) external;
    function earned(address a) external view returns (uint256);
    function getReward(address a, address recipient, bool claimExtras) external;
    function rewardToken() external view returns (address);
    function rewardRate() external view returns (uint256);
}

interface IAutopilotRouter {
    function depositMax(address autoEth, address to, uint256 minShares) external returns (uint256 shares);
}

interface IERC4626Like {
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function redeem(uint256 shares, address receiver, address owner) external;
}

/// ---------------------------
/// MYTStrategy base
/// ---------------------------
interface IMYTStrategy {
    enum RiskClass { LOW, MEDIUM, HIGH }
    struct StrategyParams {
        address owner;
        string name;
        string protocol;
        RiskClass riskClass;
        uint256 cap;
        uint256 globalCap;
        uint256 estimatedYield;
        bool additionalIncentives;
        uint256 slippageBPS;
    }

    event Allocate(uint256 amount, address strategy);
    event Deallocate(uint256 amount, address strategy);
}

contract MYTStrategy {
    IVaultV2 public immutable MYT;
    address public immutable receiptToken;
    IMYTStrategy.StrategyParams public params;
    bytes32 public immutable adapterId;
    uint256 public slippageBPS;
    address public permit2Address;

    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);
    event Allocate(uint256 amount, address strategy);
    event Deallocate(uint256 amount, address strategy);

    constructor(address _myt, IMYTStrategy.StrategyParams memory _params, address _permit2Address, address _receiptToken) {
        require(_myt != address(0), "Zero myt");
        require(_permit2Address != address(0), "Zero Permit2 address");
        require(_receiptToken != address(0), "Zero receipt token address");
        MYT = IVaultV2(_myt);
        receiptToken = _receiptToken;
        params = _params;
        adapterId = keccak256(abi.encode(_params.protocol));
        slippageBPS = _params.slippageBPS;
        permit2Address = _permit2Address;
        // Safe approve if ERC20
        try IERC20Minimal(_receiptToken).approve(_permit2Address, type(uint256).max) {} catch {}
    }

    modifier onlyVault() {
        require(msg.sender == address(MYT), "PD");
        _;
    }

    function deallocate(
        bytes memory data,
        uint256 assets,
        bytes4 selector,
        address sender
    ) external onlyVault returns (bytes32[] memory strategyIds, int256 change) {
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountDeallocated = _deallocate(assets);
        uint256 newAllocation = oldAllocation - amountDeallocated;

        emit Deallocate(amountDeallocated, address(this));

        bytes32[] memory ids_ = new bytes32[](1);
        ids_[0] = adapterId;
        return (ids_, int256(newAllocation) - int256(oldAllocation));
    }

    function _deallocate(uint256 amount) internal virtual returns (uint256) {
        revert("Not implemented");
    }
}

/// ---------------------------
/// Actual TokeAutoEthStrategy (vulnerable)
/// ---------------------------
contract TokeAutoEthStrategy is MYTStrategy {
    IERC4626Like public immutable autoEth;
    IAutopilotRouter public immutable router;
    IMainRewarder public immutable rewarder;
    address public immutable weth;
    address public immutable oracle;

    constructor(
        address _myt,
        IMYTStrategy.StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) MYTStrategy(_myt, _params, _permit2Address, _autoEth) {
        autoEth = IERC4626Like(_autoEth);
        router = IAutopilotRouter(_router);
        rewarder = IMainRewarder(_rewarder);
        weth = _weth;
        oracle = _oracle;
    }

    /// Vulnerable function — reverts here on underflow
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 sharesNeeded = autoEth.convertToShares(amount);
        uint256 actualSharesHeld = rewarder.balanceOf(address(this));

        // <-- underflow occurs here (1 - 100)
        uint256 shareDiff = actualSharesHeld - sharesNeeded;

        if (shareDiff <= 1e18) {
            sharesNeeded = actualSharesHeld;
        }

        return amount;
    }
}

/// ---------------------------
/// Mocks
/// ---------------------------
contract MockRewarder is IMainRewarder {
    uint256 public sharesHeld;
    constructor(uint256 _sharesHeld) { sharesHeld = _sharesHeld; }

    function balanceOf(address) external view override returns (uint256) { return sharesHeld; }
    function withdraw(address, uint256, bool) external override {}
    function stake(address, uint256) external override {}
    function earned(address) external pure override returns (uint256) { return 0; }
    function getReward(address, address, bool) external override {}
    function rewardToken() external pure override returns (address) { return address(0); }
    function rewardRate() external pure override returns (uint256) { return 0; }
}

contract MockAutoEth is IERC4626Like {
    uint256 public forcedShares;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor(uint256 _forcedShares) { forcedShares = _forcedShares; }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function convertToShares(uint256) external view override returns (uint256) { return forcedShares; }
    function convertToAssets(uint256 shares) external pure override returns (uint256) { return shares; }
    function redeem(uint256, address, address) external override {}
}

contract MockRouter is IAutopilotRouter {
    function depositMax(address, address, uint256) external pure override returns (uint256) { return 0; }
}

/// ---------------------------
/// PoC Test
/// ---------------------------
contract TokeAutoEthUnderflowPoC is Test {
    DummyVault public vault;
    MockERC20 public receiptToken;
    MockAutoEth public autoEth;
    MockRewarder public rewarder;
    MockRouter public router;
    TokeAutoEthStrategy public strategy;

    function setUp() public {
        receiptToken = new MockERC20("Receipt", "R");
        vault = new DummyVault(address(0xBEEF));
        autoEth = new MockAutoEth(100); // convertToShares => 100
        rewarder = new MockRewarder(1); // only holds 1 share
        router = new MockRouter();

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

        strategy = new TokeAutoEthStrategy(
            address(vault),
            params,
            address(autoEth),
            address(router),
            address(rewarder),
            address(0xDEAD),
            address(0xCAFE),
            address(0x1234)
        );
    }

    // PoC: underflow now reverts inside `_deallocate`
    function test_deallocate_underflow_reverts() public {
        bytes memory prevAllocation = abi.encode(uint256(0));
        uint256 assets = 1 ether;

        vm.prank(address(vault));
        vm.expectRevert(stdError.arithmeticError);
        strategy.deallocate(prevAllocation, assets, bytes4(0), address(this));
    }
}
```

Result:

```solidity
[PASS] test_deallocate_underflow_reverts() (gas: 29955)
Traces:
  [29955] TokeAutoEthUnderflowPoC::test_deallocate_underflow_reverts()
    ├─ [0] VM::prank(DummyVault: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  $NH{q)
    │   └─ ← [Return]
    ├─ [13712] TokeAutoEthStrategy::deallocate(0x0000000000000000000000000000000000000000000000000000000000000000, 1000000000000000000 [1e18], 0x00000000, TokeAutoEthUnderflowPoC: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ [2618] MockAutoEth::convertToShares(1000000000000000000 [1e18]) [staticcall]
    │   │   └─ ← [Return] 100
    │   ├─ [2558] MockRewarder::balanceOf(TokeAutoEthStrategy: [0xa0Cb889707d426A7A386870A03bc70d1b0697598]) [staticcall]
    │   │   └─ ← [Return] 1
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped;
```
