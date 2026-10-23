# 58639 sc medium off by one issue in the forcerepay function causes protocol to lose funds in the form of protocol fee&#x20;

**Submitted on Nov 3rd 2025 at 18:32:54 UTC by @Tarnishedx0 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58639
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

`Off by One` issue in the `_forceRepay()` function causes protocol to lose funds in the form of `protocol fee`.

## Vulnerability Details

In the `_forceRepay()` function, `protocolFeeTotal` is paid when `account.collateralBalance > protocolFeeTotal`.

```solidity
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
...
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }
...
```

But what if when`account.collateralBalance = protocolFeeTotal`.

The check should be `account.collateralBalance >= protocolFeeTotal` where `>=` should be used instead of `=`.

## Impact Details

Loss of funds for the protocol when `account.collateralBalance = protocolFeeTotal`.

## References

The above code snippets can be verified here: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L771-L775>

## Proof of Concept

## Proof of Concept

```solidity
function test_PocForceRepayOffByOne() external {
    // Set protocol fee to 1%
    vm.prank(alOwner);
    alchemist.setProtocolFee(100);

    uint256 depositAmount = 1000e18;

    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Healthy account for global collateralization
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 10);
    alchemist.deposit(depositAmount * 5, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Create test account
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Mint debt (50% of max)
    uint256 debtToMint = 500e18;
    alchemist.mint(tokenId, debtToMint, address(0xbeef));
    vm.stopPrank();

    // Create redemption for all debt
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debtToMint);
    transmuterLogic.createRedemption(debtToMint);
    vm.stopPrank();

    // Mature redemption
    vm.roll(block.number + 5_256_000);
    
    uint256 creditToYield = alchemist.convertDebtTokensToYield(debtToMint);
    uint256 targetCollateral = creditToYield / 100; // This is the protocolFeeTotal

    (uint256 currentCollateral,,) = alchemist.getCDP(tokenId);
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
    
    // Increase supply to decrease price
    uint256 priceDropRatio = (currentCollateral * 10000) / targetCollateral;
    uint256 newSupply = (initialSupply * priceDropRatio) / 10000;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);

    // Verify we achieved the condition
    alchemist.poke(tokenId);
    (uint256 exactCollateral,,) = alchemist.getCDP(tokenId);

    uint256 protocolFeeReceiverBefore = IERC20(address(vault)).balanceOf(protocolFeeReceiver);

    // Trigger liquidation with _forceRepay
    vm.startPrank(externalUser);
    alchemist.liquidate(tokenId);
    vm.stopPrank();

    uint256 protocolFeeReceiverAfter = IERC20(address(vault)).balanceOf(protocolFeeReceiver);
    uint256 feeReceived = protocolFeeReceiverAfter - protocolFeeReceiverBefore;

    // When collateralBalance == protocolFeeTotal, the bug causes protocol to receive 0
    assertEq(feeReceived, 0, "Bug: Protocol fee not transferred when collateralBalance == protocolFeeTotal");
}
```

Paste the above test in the `AlchemistV3.t.sol` contract, set up `$MAINNET_RPC_URL` and run it using:

```solidity
FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path src/test/AlchemistV3.t.sol --match-test test_PocForceRepayOffByOne -vv --evm-version cancun
```
