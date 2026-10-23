# #39539 \[SC-Insight] Insufficient validation of tokens when created in \`PlayCollateralTokenFactory::c

**Submitted on Feb 1st 2025 at 01:29:46 UTC by @Bx4 for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39539
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1-playmoney>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In PlayCollateralTokenFactory when creating a new token via `createCollateralToken` the token details are not validated this makes it easy to create duplicate tokens.

## Vulnerability Details

in the `createCollateralToken` we can see that tokens are created with specified token names, symbols and other params as seen below

```solidity
    function createCollateralToken(string memory name, string memory symbol, uint256 initialSupply, address owner)
        external
        returns (address)
    {
        PlayCollateralToken newToken = new PlayCollateralToken(name, symbol, initialSupply, CONDITIONAL_TOKENS, owner);
        emit PlayCollateralTokenCreated(address(newToken));
        return address(newToken);
    }
```

However this function does not validate the tokens being created and as a result another user can create a duplicate token of a pre-existing token.

## Impact Details

Attackers can create duplicate of token to lure users to trust duplicate token instead of original token. This makes the contract fail to return a unique list of tokens like it should.

FOR INSTANCE: An attacker could create multiple tokens with the same name and symbol using the `createCollateralToken` function. This lack of validation could lead to confusion among users, who might mistake one token for another, potentially leading to financial loss or scams.

## References

Similar vulnerabilities have been disclosed by notable firms

one example is : <https://solodit.cyfrin.io/issues/insufficient-validation-of-token-name-and-symbol-in-liquidityfreelaunchfactory-zokyo-none-zap-markdown>

## Proof of Concept

## Proof of Concept

Add this test to your foundry test environment and run. You will see that the assertion statements never fail which inidicates duplicate tokens can be created with the same name and symbol.

```solidity

    function testDuplicateTokensCreate() public {
        address owner = address(this);
        address owner1 = address(0xabc);
        uint256 supply = 120e18;
        address tokenAddr = factory.createCollateralToken("KWAMEtoken","KTN",supply, owner);
        address tokenAddr1 = factory.createCollateralToken("KWAMEtoken","KTN",supply, owner1);
        address tokenAddr2 = factory.createCollateralToken("KWAMEtoken","KTN",supply, owner);

        PlayCollateralToken token1 = PlayCollateralToken(tokenAddr);
        PlayCollateralToken token2 = PlayCollateralToken(tokenAddr1);
        PlayCollateralToken token3 = PlayCollateralToken(tokenAddr2);

        token1.transfer(address(0xabd),10e18);
        console2.log(token2.balanceOf(address(0xabd)));

        assertEq(token1.name(),"KWAMEtoken");
        assertEq(token2.name(), "KWAMEtoken");
        assertEq(token3.name(),"KWAMEtoken");
        assertEq(token2.symbol(),"KTN");
        assertEq(token1.symbol(),"KTN");
        assertEq(token3.symbol(),"KTN");
    }
```
