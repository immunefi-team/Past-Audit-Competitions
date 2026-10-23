# Boost \_ Folks Finance 33675 - \[Smart Contract - Low] PythNodeprocess can revert because of incorrect

Submitted on Fri Jul 26 2024 06:49:20 GMT-0400 (Atlantic Standard Time) by @OxAnmol for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33675

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

If the `pyth` oracle returns the exponent < -18 then the `factor.toUint256()` will revert due to the incorrect casting.

## Vulnerability Details

```solidity
 function process(bytes memory parameters) internal view returns (NodeOutput.Data memory nodeOutput) {
        (address pythAddress, bytes32 priceFeedId, bool useEma) = abi.decode(parameters, (address, bytes32, bool));

        /// @dev using unsafe methods to avoid reverting, so this accepts old data
        IPyth pyth = IPyth(pythAddress);
        PythStructs.Price memory pythData = useEma
            ? pyth.getEmaPriceUnsafe(priceFeedId)
            : pyth.getPriceUnsafe(priceFeedId);

        /// @dev adjust the price to 18 d.p., exponent is a int32 so it could be negative or positive
        int256 factor = PRECISION + pythData.expo; // 1e18 + -8 = 1e10
        uint256 price = factor > 0
            ? pythData.price.toUint256() * (10 ** factor.toUint256())
->>          : pythData.price.toUint256() / (10 ** factor.toUint256());

        return NodeOutput.Data(price, pythData.publishTime, NodeDefinition.NodeType.PYTH, 0, 0);
    }
```

Here the code expects the factor to be ≥0, but in case the `pythData.expo` is < -18 in that case the factor will be negative, and if it is negative the `toUint256` of `SafeCast` will revert as you can see in the code.

```solidity
 function toUint256(int256 value) internal pure returns (uint256) {
       //Revert if negative
        if (value < 0) {
            revert SafeCastOverflowedIntToUint(value);
        }
        return uint256(value);
    }
```

Here is how the original implementation from the synthetic that handles this issue. <https://github.com/Synthetixio/synthetix-v3/blob/8aff01938913983b97faa5ce082c15b86db32e0d/protocol/oracle-manager/contracts/nodes/pyth/PythNode.sol#L32>

## Impact Details

The pyth priceFeed where the exponent is < -18 can cause the malfunctioning of the protocol.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/PythNode.sol#L36>

## Proof of concept

This is a test from `PythNode.test.ts`, here you can see that the test reverts if we override `decimals` from 8 to 20.

```js
it.only("Should process price correctly with precision smaller than exponent", async function () {
      const nodeOutput = await nodeManager.process(nodeId);
      expect(nodeOutput.price).to.equal(ethers.parseUnits(price.toString(), PRECISION - decimals));
      expect(nodeOutput.timestamp).to.equal(updateTimestamp);
      expect(nodeOutput.additionalParam1).to.equal(0);
      expect(nodeOutput.additionalParam2).to.equal(0);
    });
```
