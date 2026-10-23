# 28942 - \[SC - Insight] Self Destruction of inchRouter can lead to loss...

Submitted on Mar 2nd 2024 at 14:39:01 UTC by @oxumarkhatab for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28942

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Due to having a hardcoded address of 1inch AggregatorRouterV5 in the Puffer Depositor contract, and AggregatorRouterV5 having a self-destruct function in its implementation, the protocol's 1inch Swap's functionality will be broken.

Having a hardcoded immutable address of the Router escalates the issue because the immutable variables are part of the bytecode and will never get updated.

## Why I have chosen the selected impact

I was not able to find an impact that suits my needs of the impact because this vulnerability greatly affects the entire protocol, so griefing was the closest one not due to its impact, because the impact of this vulnerability is High severity in my opinion.

## Vulnerability Details

Offer a detailed explanation of the vulnerability itself. Do not leave out any relevant information. Code snippets should be supplied whenever helpful, as long as they don’t overcrowd the report with unnecessary details. This section should make it obvious that you understand exactly what you’re talking about, and more importantly, it should be clear by this point that the vulnerability does exist.

## Impact Details

The Puffer depositor smart contract relies on AggregatorV5 Router for doing 1Inch Swaps.

```solidity
   function swapAndDeposit1Inch(address tokenIn, uint256 amountIn, bytes calldata callData)
        public
        payable
        virtual
        restricted
        returns (uint256 pufETHAmount)
    {
        if (tokenIn != _NATIVE_ETH) {
            SafeERC20.safeTransferFrom(IERC20(tokenIn), msg.sender, address(this), amountIn);
            SafeERC20.safeIncreaseAllowance(IERC20(tokenIn), address(_1INCH_ROUTER), amountIn);
        }

        // PUFFER_VAULT.deposit will revert if we get no stETH from this contract
        (bool success, bytes memory returnData) = _1INCH_ROUTER.call{ value: msg.value }(callData);
        if (!success) {
            revert SwapFailed(address(tokenIn), amountIn);
        }

        uint256 amountOut = abi.decode(returnData, (uint256));

        if (amountOut == 0) {
            revert SwapFailed(address(tokenIn), amountIn);
        }

        return PUFFER_VAULT.deposit(amountOut, msg.sender);
    }


```

In this function , take a look at this line :

```solidity
_1INCH_ROUTER.call{ value: msg.value }(callData);
```

The `_1INCH_ROUTER` is an immutable variable having the 1InchRouter address

`address internal constant _1INCH_ROUTER = 0x1111111254EEB25477B68fb85Ed929f73A960582;`

which means it can not be changed later . Things are alright since here but The problem arises when we look at the code of 1InchRouter's implementation.

```solidity


/// @notice Main contract incorporates a number of routers to perform swaps and limit orders protocol to fill limit orders
contract AggregationRouterV5 is EIP712("1inch Aggregation Router", "5"), Ownable,
    ClipperRouter, GenericRouter, UnoswapRouter, UnoswapV3Router, OrderMixin, OrderRFQMixin
{
    using UniERC20 for IERC20;

    error ZeroAddress();

    /**
     * @dev Sets the wrapped eth token and clipper exhange interface
     * Both values are immutable: they can only be set once during
     * construction.
     */
    constructor(IWETH weth)
        UnoswapV3Router(weth)
        ClipperRouter(weth)
        OrderMixin(weth)
        OrderRFQMixin(weth)
    {
        if (address(weth) == address(0)) revert ZeroAddress();
    }

    /**
     * @notice Retrieves funds accidently sent directly to the contract address
     * @param token ERC20 token to retrieve
     * @param amount amount to retrieve
     */
    function rescueFunds(IERC20 token, uint256 amount) external onlyOwner {
        token.uniTransfer(payable(msg.sender), amount);
    }

    /**
     * @notice Destroys the contract and sends eth to sender. Use with caution.
     * The only case when the use of the method is justified is if there is an exploit found.
     * And the damage from the exploit is greater than from just an urgent contract change.
     */
    function destroy() external onlyOwner {
        selfdestruct(payable(msg.sender));
    }

    function _receive() internal override(EthReceiver, OnlyWethReceiver) {
        EthReceiver._receive();
    }
}
```

Take a look at `destroy` method:

`function destroy() external onlyOwner { selfdestruct(payable(msg.sender)); }`

although it is owner-only there is a chance that the protocol is attacked where `damage from the exploit is greater than from just an urgent contract change.` as stated in its NetSpec.

When the protocol is self-destructed, the calls to the 1inchRouter will always fail due to which the entire functionality of the Puffer depositor will be jammed and protocol will not work as intended

## References

1Inch Router (scroll down to see destroy function ):

<https://etherscan.io/address/0x1111111254EEB25477B68fb85Ed929f73A960582#code>

## Proof of Concept

Even the biggest protocols like `kyberswap` are hacked due to the `0.0000001%` chance of it being exploited so there is a chance that 1InchRouter is also hacked, `self-destructed` to prevent potentially greater damages, and be `deployed at a new address` which now new users can use. However, our current implementation does not allow to change the deployed address of 1inchRouter without revamping and re-deploying the entire depositor contract and other contracts that are harcodedly connected with the address of the puffer depositor.

This Vulnerability is clear that self-destruct can remove the 1inchRouter from the blockchain which will make all the forwarded calls to it, fail. Hence I was not able to practically self-destruct the 1inchRouter to show my PoC : )
