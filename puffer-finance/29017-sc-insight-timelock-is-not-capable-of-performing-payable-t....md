# 29017 - \[SC - Insight] Timelock is not capable of performing payable t...

Submitted on Mar 4th 2024 at 19:46:44 UTC by @oxumarkhatab for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29017

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Timelock will not be able to execute payable transactions and it will always fail because it does not execute transactions with funds nor does it keep the record of msg.value per transaction when it is queued. This will cause protocol operation issues as well as loss of funds for users.

Timelock being an upgradable contract was another potential way to add new functionality to the contract but it is non-upgradable contract if we see it's declaration leaving itself prone to the issues I'll be describing in the report

## Impact Selection

I've selected this impact because it was closest to this exploit's impact.

The timelock intends to execute all types of transactions but due to programming oversight, it is not able to do so.

That's why I've chosen a similar impact

Additionally, users will also lose funds , let's see in detail

## Vulnerability Details

Each transaction is time-locked including payable and non-payable transactions. While Timelock appears to deal greatly with non-payable transactions, It will never be able to execute payable transactions due to the way it executes the transaction.

See here :

```solidity
    function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        // slither-disable-next-line arbitrary-send-eth
        return target.call(callData);
    }

```

This function is the core part of timelock where the actual transaction happens. See closely that the function is making a static call ( which is fine ) but it is not sending any msg.value along with the call that will cause following functions to revert

```solidity
    function swapAndDeposit(address tokenIn, uint256 amountIn, uint256 amountOutMin, bytes calldata routeCode)
        public
        payable
        virtual
        restricted
        returns (uint256 pufETHAmount)
    {
        if (tokenIn != _NATIVE_ETH) {
            SafeERC20.safeTransferFrom(IERC20(tokenIn), msg.sender, address(this), amountIn);
            SafeERC20.safeIncreaseAllowance(IERC20(tokenIn), address(_SUSHI_ROUTER), amountIn);
        }

        uint256 stETHAmountOut = _SUSHI_ROUTER.processRoute{ value: msg.value }({
            tokenIn: tokenIn,
            amountIn: amountIn,
            tokenOut: address(_ST_ETH),
            amountOutMin: amountOutMin,
            to: address(this),
            route: routeCode
        });

        if (stETHAmountOut == 0) {
            revert SwapFailed(address(tokenIn), amountIn);
        }

        return PUFFER_VAULT.deposit(stETHAmountOut, msg.sender);
    }

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

because they expect msg.value to be non-zero `when swaping NATIVE_ETH` token.

The root of the issue is qeueTransaction function which actually registers a transaction as pending one to be executed after delay :

```solidity

    function queueTransaction(address target, bytes memory callData, uint256 operationId) public returns (bytes32) {
        if (msg.sender != OPERATIONS_MULTISIG) {
            revert Unauthorized();
        }

        bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
        uint256 lockedUntil = block.timestamp + delay;
        if (queue[txHash] != 0) {
            revert InvalidTransaction(txHash);
        }
        queue[txHash] = lockedUntil;
        // solhint-disable-next-line func-named-parameters
        emit TransactionQueued(txHash, target, callData, operationId, lockedUntil);

        return txHash;
    }
```

This neither receives any ether ( because it is not a payable function ) nor it record keep the amount of ether the function call expects.

Additionally, the entire structure seems a bit off. Queue transaction is only callable by operations multisig then to whom should the user send the msg.value .

If funds are sent to timelock , and execute Transaction fails due to lack of funds , the timelock contract does not even have any withdraw or redeem function to recover those funds of users `causing Direct theft of user funds`

Another scenario might be that funds are sent to PufferDepositor contract. No matter how much funds PufferDepositor has , the swap calls just check msg.value and not address(this).

In each way , the logic of timelock contract is bricked which should be corrected.

## Impact Details

* Protocol can not perform Deposits involving ethers
* Users will lose their funds when they send funds for payable transactions execution because payable transactions will not be successful and all users will get is a boolean of `success` which has a value of `false`

## References

Timelock contract : Line 263

<https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA?utm\\_source=immunefi#code>

## Proof of Concept

Consider following scenario :

* User makes a transaction object with

```
target:address(PufferDepositor),
calldata: some_call_Data_for_swap like following

 bytes memory _1inchSwap_calldata = abi.encode(
        // insert 1inch swap calldata here where we specify amountIn and src and destination.
        // specify src=PufferDepositor because it will be swapping on our behalf
        // specify dest as pufferDepositor

     );

tokenIn:NATIVE_ETHER,
amountIn:100 ether
```

* User sends 100 ether to timelock.
* Operations Multisig calls `queueTransaction` and adds transaction to be executed.
* After some delay , the OPERATIONS\_MULTISIG calls executeTransaction function.
* The function returns a success value of false.
* User sees that he has not given any shares ( because transaction was failed )
* User tries to recover funds but timelock does not provide any ways to recover funds.
