# 30939 - \[SC - Critical] Misuse of curve pool calls results for precisio...

Submitted on May 8th 2024 at 16:46:08 UTC by @OceanAndThunders for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30939

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Protocol insolvency
* Contract fails to deliver promised returns, but doesn't lose value

## Description

Hello team,

## Brief/Intro

**Explaining the curvePool's exchange\_underlying logic** The pool adapter contracts (CurveMetaPoolAdapter) uses the curve pool to melt and swap tokens, however the understanding of exchange and exchange\_underlying calls are not being properly used and understood which allows the protocol vulnerable to precesion loss (which leads to successful sandwich attacks against the RevenueHandler contract) and unintended revert calls which user's transactions will fails to deliver promised returns

## Vulnerability Details

The curve pool contracts uses the famous "minimum amount to receive" argument for swap (exchange/exchange\_underlying functions) calls, this option allows a user who tries to exchange userA for userB to specify the minimum amount of tokenB to receive, and if the amount is under the swap shall revert, this mechanism will prevent front running hacks, and sandwich attacks

see the curve pool exchange\_underlying function (etherscan.io/address/0x890f4e345B1dAED0367A877a1612f86A1f86985f)

```
@external
@nonreentrant('lock')
def exchange_underlying(i: int128, j: int128, dx: uint256, min_dy: uint256) -> uint256:
    """
    @notice Perform an exchange between two underlying coins
    @dev Index values can be found via the `underlying_coins` public getter method
    @param i Index value for the underlying coin to send
    @param j Index valie of the underlying coin to recieve
    @param dx Amount of `i` being exchanged
    @param min_dy Minimum amount of `j` to receive
    @return Actual amount of `j` received
```

While the forth arg (min\_dy) is the minimum amount of `j` to receive of underlying token to get

After the swap is done, it will compare the tokens converted to min\_dy, and it shall reverts as I explained previously :

```
    assert dy >= min_dy, "Too few coins in result"
```

You can read more at : <https://curve.readthedocs.io/exchange-pools.html>

```
Perform an exchange between two coins.

i: Index value for the coin to send

j: Index value of the coin to receive

_dx: Amount of i being exchanged

_min_dy: Minimum amount of j to receive

Returns the actual amount of coin j received. Index values can be found via the coins public getter method.

expected = pool.get_dy(0, 1, 10**2) * 0.99
pool.exchange(0, 1, 10**2, expected, {"from": alice})
```

***

***

**The issue**

While the amount of underlying token to send is likely to be much higher or lower than the amount of underlying token to receive, The 'RevenueHandler' however sets the amount to send is as the same as the amount to receive (see : <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L288>) line 292, 293

```
  return
            IPoolAdapter(poolAdapter).melt(
                revenueToken,
                tokenConfig.debtToken,
                revenueTokenBalance,
                revenueTokenBalance
            );
```

and the poolAdapter's melt is as :

```
  function melt(
        address inputToken,
        address outputToken,
        uint256 inputAmount,
        uint256 minimumAmountOut
    ) external override returns (uint256) {
        TokenUtils.safeApprove(inputToken, pool, inputAmount);
        return
            ICurveMetaSwap(pool).exchange_underlying(
                tokenIds[inputToken],
                tokenIds[outputToken],
                inputAmount,
                minimumAmountOut,
                msg.sender
            );
```

Input amount is the amount of underlying token to send and the minimumAmountOut is the expected amount in to receive to prevent front running and sandwich attacks

Setting the Input amount to send the same as minimumAmountOut is a huge mistake, as will actually results for these two cases :

**Case1/** : tokenA's value is higher than tokenB's value, so 100 tokenA equal to 1000 tokenB,

> so here when 100 is set as inputAmount and minimumAmountOut, will not actually prevent sandwich attack and will make the "RevenueHandler" contract be a victim of sandwich/frontrunning attack as the slippage is very low (sending 100 and expecting 1000 as basic, however the minimumAmountOut indecates that it's okat to receive any amount higher than 100 !)

**Case2/** : tokenA's value is lower than tokenB's value, so 100 tokenA equal to 50 tokenB,

> so here when 100 is set as inputAmount and minimumAmountOut, will make the call :

```
RevenueHandler>checkpoint>_melt>CurveMetaPoolAdapter.melt>curvePool.exchange_underlying
```

reverts with "Too few coins in result" as the revenue contract was is supposed to get 50 tokenB but it set it's "minimumAmountOut" to 100, this will makes the checkpoint call always reverts, and thus the contract will loses it's value permanently (since the minimumAmountOut is set by default and not controlled nor edited) !

## Impact Details

Setting the Input amount to send the same as minimumAmountOut is a huge mistake, as will actually results for these two cases :

**Case1/** : tokenA's value is higher than tokenB's value, so 100 tokenA equal to 1000 tokenB,

> so here when 100 is set as inputAmount and minimumAmountOut, will not actually prevent sandwich attack and will make the "RevenueHandler" contract be a victim of sandwich/frontrunning attack as the slippage is very low (sending 100 and expecting 1000 as basic, however the minimumAmountOut indecates that it's okat to receive any amount higher than 100 !)

**Case2/** : tokenA's value is lower than tokenB's value, so 100 tokenA equal to 50 tokenB,

> so here when 100 is set as inputAmount and minimumAmountOut, will make the call :

```
RevenueHandler>checkpoint>_melt>CurveMetaPoolAdapter.melt>curvePool.exchange_underlying
```

reverts with "Too few coins in result" as the revenue contract was is supposed to get 50 tokenB but it set it's "minimumAmountOut" to 100, this will makes the checkpoint call always reverts, and thus the contract will loses it's value permanently (since the minimumAmountOut is set by default and not controlled nor edited) !

## References

<https://curve.readthedocs.io/exchange-pools.html>

## Proof of Concept

checking both cases for "0x890f4e345B1dAED0367A877a1612f86A1f86985f" and "0xa5407eae9ba41422680e2e00537571bcc53efbfd" ....etc and all curve pools

get\_dy\_underlying function checks how much a user shall receive for the exchange 1000 token for i against j !

Used truffle and ganache :

```
ganache-cli --fork https://mainnet.infura.io/v3/ef9488938aac4f0e9d3f755bb20b2cc0 --networkId 5777
```

```
truffle console --networkId 5777
```

Enter the following commands one by one :

```
// copy the "0x890f4e345B1dAED0367A877a1612f86A1f86985f" contract abi at /test/abi.json (implementation of 0xba7d1581Db6248DC9177466a328BF457703c8f84)
const abi = require('./test/abi.json');

const v_ = "0x890f4e345B1dAED0367A877a1612f86A1f86985f"
const Web3 = require('web3');
const vc = new web3.eth.Contract(abi,v_);
await vc.methods.get_dy_underlying(0,1,1000).call()
//will returns
//'20'
```

so for that case when the InputAmount is the same as minAmountOut (as 1000 both) will make the call reverts with "Too few coins in result" as the revenue contract was is supposed to get 20 on token J but it set it's "minimumAmountOut" to 1000 !

case 2, make this call on the same session

```
await vc.methods.get_dy_underlying(1,0,1000).call()
//will returns '47211'
```

so for that case when the InputAmount is the same as minAmountOut (as 1000 both) will make the call RevenueHandler contract prone to sandwich/front run attack, as the revenue contract was is supposed to get 47211 on token J but it set it's "minimumAmountOut" to 1000 ! this is a very high slippage rate, when such a call is made the revenueHandler contract is prone to lose 460% of it's return by front running/sandwich attacks

Regards,

Adam
