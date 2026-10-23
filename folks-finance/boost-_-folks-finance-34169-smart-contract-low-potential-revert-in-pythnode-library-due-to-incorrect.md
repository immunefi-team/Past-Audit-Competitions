# Boost \_ Folks Finance 34169 - \[Smart Contract - Low] Potential revert in PythNode library due to inc

Submitted on Tue Aug 06 2024 04:44:59 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34169

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

In the `PythNode` library's `process` function, there's a potential issue with the price calculation logic when dealing with negative exponents from Pyth price feeds. The function attempts to convert a potentially negative `int256` value to `uint256` using the `SafeCast.toUint256()` method, which will always revert for negative inputs.

See: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/nodes/PythNode.sol>

```solidity
uint256 price = factor > 0
    ? pythData.price.toUint256() * (10 ** factor.toUint256())
    : pythData.price.toUint256() / (10 ** factor.toUint256());
```

When `factor` is negative, the second branch is executed, attempting to convert `factor` to `uint256`. This conversion will invariably fail for negative values, causing the transaction to revert.

See: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeCast.sol#L567-L579>

**Note**: `PythNode` library is a part of `NodeManager.sol` (in-scope asset).

## Impact

The impact of this issue is that it will cause a revert in the `process` function when dealing with price feeds that result in a negative `factor`.

## Rationale for Severity

The severity of this issue is considered **Low** for the following reasons:

1. This branch might be unreachable in the first place as it requires < -18 exponent return from a Pyth node, which is currently non-existent.
2. The affected price feeds might not be integrated with the protocol.

Despite the two reasons mentioned, it's still a techically valid bug and should be fixed.

## Proof of concept

## Proof-of-Concept

### Requirement

* chisel 0.2.0 (3ae4f50 2024-07-07T00:21:48.451168337Z)

### Steps

1. Run `git clone https://github.com/Folks-Finance/folks-finance-xchain-contracts.git`
2. Run `cd folks-finance-xchain-contracts`
3. Run `chisel` inside the directory
4. Run below code, **line by line** in the REPL

```
import "@openzeppelin/contracts/utils/math/SafeCast.sol";
using SafeCast for int256;
int256 factor = -1;
factor.toUint256();
```

5. Observe that it reverts when trying to convert `factor` to `uint256`
