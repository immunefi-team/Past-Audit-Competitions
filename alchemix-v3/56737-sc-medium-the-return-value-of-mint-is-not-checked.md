# 56737 sc medium the return value of mint is not checked

**Submitted on Oct 20th 2025 at 04:25:20 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56737
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The MoonwellUSDCStrategy.sol::\_allocate() invoke mUSDC::mint() to allocate assets , however the return value of it is not checked

## Vulnerability Details

In the MoonwellUSDCStrategy.sol::\_allocate()

```solidity
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
        // Mint mUSDC with underlying USDC
        mUSDC.mint(amount); //@audit return value is not checked.
        return amount;
    }
```

Assets are first transferred from the Morpho V2 Vault, and then `MYTStrategy.sol::deallocate()` is invoked. If the mint operation fails silently (i.e., without reverting), the transferred assets can become stuck in the contract.

## Impact Details

1.assets become stuck in the contract 2.Those assets are not included in the `newTotalAssets` calculation within the Morpho V2 Vault

## References

## Proof of Concept

## Proof of Concept

Frist we can find the address of mUSDC on op network via this link <https://docs.moonwell.fi/moonwell/protocol-information/contracts#op-mainnet-contract-addresses>

The address is : <https://optimistic.etherscan.io/address/0x8E08617b0d66359D73Aa11E11017834C29155525>

the mint function :

```solidity

    /**
     * @notice Sender supplies assets into the market and receives mTokens in exchange
     * @dev Accrues interest whether or not the operation succeeds, unless reverted
     * @param mintAmount The amount of the underlying asset to supply
     * @return uint 0=success, otherwise a failure (see ErrorReporter.sol for details)
     */
    function mint(uint mintAmount) external override returns (uint) {
        (uint err, ) = mintInternal(mintAmount);
        return err;
    }


    function mintInternal(
        uint mintAmount
    ) internal nonReentrant returns (uint, uint) {
        uint error = accrueInterest();
        if (error != uint(Error.NO_ERROR)) {
            // accrueInterest emits logs on errors, but we still want to log the fact that an attempted borrow failed
            return (
                fail(Error(error), FailureInfo.MINT_ACCRUE_INTEREST_FAILED),
                0
            );
        }
        // mintFresh emits the actual Mint event if successful and logs on errors, so we don't need to
        return mintFresh(msg.sender, mintAmount);
    }
```

**@return uint 0=success, otherwise a failure (see ErrorReporter.sol for details)**
