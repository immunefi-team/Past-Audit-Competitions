# Boost \_ Folks Finance 33953 - \[Smart Contract - Low] Calling process function will not revert even i

Submitted on Fri Aug 02 2024 16:08:08 GMT-0400 (Atlantic Standard Time) by @bbl4de for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33953

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `PriceDeviationSameOracleCircuitBreakerNode` library is responsible to determine the price deviation of the prices provided by the two main oracle nodes AND ensure these oracles are not of the same type. Unfortunately, specifying three parent nodes for proper price deviation handling, the `process()` function from mentioned library fails to check the type of used oracles is not the same - failing to deliver promised results.

## Vulnerability Details

When we ensure that we're safe from price deviation - by having the third price source in `parentNodeOutputs` ( at index 2 ):

```
if (percentageDifference > deviationTolerance) {
            if (parentNodeOutputs.length == 2) revert DeviationToleranceExceeded(percentageDifference);
            return parentNodeOutputs[2];
        }
```

We simultaneously make the following check to NEVER evaluate to *true*, because `parentNodeOutputs.length != 2` :

```
if (parentNodeOutputs.length == 2 && parentNodeOutputs[0].nodeType == parentNodeOutputs[1].nodeType) {
            revert SameOracle(parentNodeOutputs[0].nodeType);
        }
```

Essentially skipping the same oracle type check.

## Impact Details

It causes the oracle adapter to work in an unintended way, making it possible for the returned price to be from one of two oracles of the same type. This means that it fails to deliver promised results as it could mean two oracle nodes of the same type are used, increasing the risk of price manipulation and breaking price feed safety assurances of the protocol.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/PriceDeviationSameOracleCircuitBreakerNode.sol#L35-L41>

## Proof of concept

## Proof of Concept

To prove the node will not revert with SameOracle error when it should, make the following change to `PriceDeviationSameOracleCircuitBreakerNode.test.ts`:

1. In the `describe` method "Contract methods" and `it` method "Should process correctly" test case, change the `parentNodeIds` to this, where two of the first nodes have the same type:

```
        parentNodeIds = [external22NodeId, external22NodeId, constant69NodeId];
```

link to this test: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/test/oracle/nodes/PriceDeviationSameOracleCircuitBreakerNode.test.ts#L108>

2. To run this test case individually add `.only` method to the `it` statement:

```
it.only("Should process correctly", async function () {
```

3. Run the test with:

```
npm test
```

As we can see, when the price deviation is within bounds, the node does not revert even if the first two nodes are of the same type. To verify that it works also for two different nodes of the same type, change `parentNodeIds` to:

```
        parentNodeIds = [constant42NodeId, constant69NodeId, external22NodeId];

```

and check the last `expect` statement:

```diff
-        expect(nodeOutput.price).to.equal(externalNodePrice);
+        expect(nodeOutput.price).to.equal(constant42NodePrice);
```

and run the test again - it will also pass without reverting.
