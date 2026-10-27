# 56801 sc insight function burn could be gas optimized

**Submitted on Oct 20th 2025 at 20:16:30 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56801
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `burn` uses the same calculations redundantly, these calculations could be gas optimized by creating one variable.

## Vulnerability Details

Code snippet:

```solidity
        // Debt is subject to protocol fee similar to redemptions
        _accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;

        TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
        _mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;
```

As seen in the code, the function performs the same calculation three times:

```solidity
convertDebtTokensToYield(credit) * protocolFee / BPS;
```

This could be optimized by simply creating a uint256 variable `optimizedAmount` and using this variable instead of calculating each time. This means the calculation would be needed just once. Which saves significant gas.

## Impact Details

This is a gas optimization. This would save roughly 45,000 gas per call. Which is significant.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L483>

## Proof of Concept

## Proof of Concept

Add this function in the `AlchemistV3.t.sol` test file:

```solidity
        function test_MyBestPoC() external {
        uint256 amount = 100e18;

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));

        vm.roll(block.number + 1);

        SafeERC20.safeApprove(address(alToken), address(alchemist), amount / 2);
        alchemist.burn(amount / 2, tokenId);
        vm.stopPrank();

        (, uint256 userDebt,) = alchemist.getCDP(tokenId);

        assertEq(userDebt, 0);
    }
```

Running this function without any optimization shows cost of 829412 gas:

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_MyBestPoC() (gas: 829412)
Suite result: ok. 1 passed; 0 failed; 0 skipped;
```

Now, let's optimize the function `burn`: Optimized version:

```solidity
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) { // ✅
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientId);
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == _accounts[recipientId].lastMintBlock) revert CannotRepayOnMintBlock();

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before more is taken
        _sync(recipientId);

        uint256 debt;
        // Burning alAssets can only repay unearmarked debt
        _checkState((debt = _accounts[recipientId].debt - _accounts[recipientId].earmarked) > 0);

        uint256 credit = amount > debt ? debt : amount;

        // Must only burn enough tokens that the transmuter positions can still be fulfilled
        if (credit > totalSyntheticsIssued - ITransmuter(transmuter).totalLocked()) {
            revert BurnLimitExceeded(credit, totalSyntheticsIssued - ITransmuter(transmuter).totalLocked());
        }

        // Burn the tokens from the message sender
        TokenUtils.safeBurnFrom(debtToken, msg.sender, credit);

        uint256 cachedAmount = convertDebtTokensToYield(credit) * protocolFee / BPS;
        // Debt is subject to protocol fee similar to redemptions
        _accounts[recipientId].collateralBalance -= cachedAmount;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, cachedAmount);
        _mytSharesDeposited -= cachedAmount;
        // Update the recipient's debt.
        _subDebt(recipientId, credit);

        totalSyntheticsIssued -= credit;

        emit Burn(msg.sender, credit, recipientId);

        return credit;
    }
```

Result after optimization:

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_MyBestPoC() (gas: 784684)
Suite result: ok. 1 passed; 0 failed; 0 skipped;
```
