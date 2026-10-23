# 28813 - \[SC - Insight] PufferVaultclaimWithdrawalFromLido according to...

Submitted on Feb 27th 2024 at 23:19:21 UTC by @ihtishamsudo for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28813

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`claimWithdrawalFromEigenLayer` should be restricted access, according to natspec but unlike other functions it has not implemented `restricted` function

## Vulnerability Details

`PufferVault::claimWithdrawalFromEigenLayer` function is used to claim `stETH` withdrawals from `Eigen Layer` and it netspac indicates that it should be `restricted access` as shown below

```
    /**
           * @notice Claims stETH withdrawals from EigenLayer
@>         * Restricted access
           * @param queuedWithdrawal The queued withdrawal details
           * @param tokens The tokens to be withdrawn
           * @param middlewareTimesIndex The index of middleware times
     */
```

But it does not implement `restricted` modifier, whereas every other function having restricted access in natspec implemented the restricted function clearly but it clearly doesn't implement.

## Impact Details

As far as the `netspac` indicates that it should be restricted access so only restricted address can call it but having no restricted modifer means anyone can call this function.

## References

<https://github.com/PufferFinance/pufETH/blob/d340d40a2ebb72993cd7dd6049a78a01bcef32ae/src/PufferVault.sol#L217>

<https://github.com/PufferFinance/pufETH/blob/d340d40a2ebb72993cd7dd6049a78a01bcef32ae/src/PufferVault.sol#L226>

## Proof of Concept

Here is the test in `PufferTest.integration.t.sol` that is testing withdrawal from Eigen layer and this test is using `OPERATION_MULTISIG` address to call every function in this test and `OPERATION_MULTISIG` is one of the puffer team address.

But what if we mold the test to let any random address to call claim withdrawal. For this we have to let `OPERATION_MULTISIG` stop prank and prank it with a random address right before claim withdrawal.

```solidity
function test_withdraw_from_eigenLayer()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), address(pufferVault), 1000 ether) // Blast got a lot of stETH
    {
        // Simulate stETH cap increase call on EL
        _increaseELstETHCap();

@>        vm.startPrank(OPERATIONS_MULTISIG); //normal called by multisig (restricted address) puffer team address
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
@>  vm.stopPrank()     //stop `OPEATION_MULTISIG` call       
        // Roll block number + 100k blocks into the future
        vm.roll(block.number + 100000);
@>  vm.prank(address(5)) // Pranked with a random address and test it
        // Claim Withdrawal
        pufferVault.claimWithdrawalFromEigenLayer(queuedWithdrawal, tokens, 0);

        // 1 wei diff because of rounding
        assertApproxEqAbs(assetsBefore, pufferVault.totalAssets(), 1, "should remain the same after withdrawal");
    }
```

#### And the test passes

```solidity
Running 2 tests for test/Integration/PufferTest.integration.t.sol:PufferTest
[PASS] test_withdraw_from_eigenLayer() (gas: 660064)
[PASS] test_withdraw_from_eigenLayer_dos() (gas: 888388)
Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.34s
 
Ran 1 test suites: 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

If the proper restricted modifier was implemented correctly than this test should've been failed but lack of it let any random address to call this function.
