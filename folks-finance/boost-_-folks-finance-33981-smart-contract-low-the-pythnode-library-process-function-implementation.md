# Boost \_ Folks Finance 33981 - \[Smart Contract - Low] The PythNode library process function implement

Submitted on Sat Aug 03 2024 07:13:32 GMT-0400 (Atlantic Standard Time) by @bbl4de for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33981

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `PythNode` is supposed to support tokens with arbitrary decimals. This is not the case when the returned price's exponent is greater than `PRECISION` - when the token has more than 18 decimals.

## Vulnerability Details

`PythNode` implements the `process()` function, where the following code is responsible for adjusting decimals:

```
/// @dev adjust the price to 18 d.p., exponent is a int32 so it could be negative or positive
        int256 factor = PRECISION + pythData.expo;
        uint256 price = factor > 0
            ? pythData.price.toUint256() * (10 ** factor.toUint256())
            : pythData.price.toUint256() / (10 ** factor.toUint256());
```

Although it takes into account that the `factor` variable may be negative, it will revert in this situation because of the check in the `SafeCast` library's `toUint256()` function:

```
  /**
     * @dev Converts a signed int256 into an unsigned uint256.
     *
     * Requirements:
     *
     * - input must be greater than or equal to 0.
     */
function toUint256(int256 value) internal pure returns (uint256) {
        if (value < 0) {
@>          revert SafeCastOverflowedIntToUint(value);
        }
        return uint256(value);
    }
```

as we can see in the code and comment above this library function does NOT support casting from a negative int256.

## Impact Details

When the `PythNode` is used, it is assumed that it supports tokens with any decimals. However, because of the misuse of the SafeCast library it is not the case and requesting a price for any token with >18 decimals from this node will revert.

## References

`PythNode::process()`:

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/PythNode.sol#L33-L36>

`SafeCast::toUint256()`:

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/24a641d9c9e0137093592a466c5496315626d98d/contracts/utils/math/SafeCast.sol#L574-L579>

## Proof of concept

## Proof of Concept

In `PythNode.test.ts` at the following code here <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/test/oracle/nodes/PythNode.test.ts#L119> :

```
it.only("Should process price correctly with precision smaller than exponent", async function () {
      const nodeOutput = nodeManager.process(nodeId);
      const factor = -2;
      await expect(nodeOutput)
        .to.be.revertedWithCustomError(nodeManager, "SafeCastOverflowedIntToUint")
        .withArgs(factor);
    });
```

To verify the issue is valid without having to make major changes to the mocks and helper files in the codebase, we can just change the `PRECISION` value in `PythNode.sol` to `6`:

```diff
- int256 public constant PRECISION = 18;
+ int256 public constant PRECISION = 6;
```

This is because the decimals used for the mock price is `8`. To verify that the call reverts with custom error `SafeCastOverflowedIntToUint`, run the test with:

```
npm test
```

Although this PoC requires changing `PythNode.sol` it's exactly the same as if the price had 20 decimals and the `PRECISION` was 18 as it is. The intention of this PoC is to visualize the obvious revert from the SafeCast library and so changes required were minimized. The issue itself lays in choosing tokens with large decimals - which would trigger the same error as the test case above.
