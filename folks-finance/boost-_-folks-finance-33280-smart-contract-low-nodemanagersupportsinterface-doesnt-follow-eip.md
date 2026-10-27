# Boost \_ Folks Finance 33280 - \[Smart Contract - Low] NodeManagersupportsInterface doesnt follow EIP-

Submitted on Tue Jul 16 2024 23:29:16 GMT-0400 (Atlantic Standard Time) by @chista0x for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33280

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `NodeManager` contract incorrectly implements the `supportsInterface` function, which fails to comply with the EIP-165 standard. As a result, it incorrectly returns `false` for the EIP-165 interface ID `0x01ffc9a7`. This could lead to compatibility issues with other contracts and systems that rely on correct EIP-165 interface support.

## Vulnerability Details:

The `NodeManager` contract implements the `INodeManager` interface, which extends `IERC165`, and is therefore expected to comply with EIP-165. According to EIP-165, the `supportsInterface` function should return:

* `true` for the `0x01ffc9a7` (EIP-165 interface ID)
* `false` for the `0xffffffff`
* `true` for any other `interfaceID` this contract implements
* `false` for any other `interfaceID`

However, `NodeManager.supportsInterface()` currently returns `false` for `0x01ffc9a7`, which is incorrect.

The relevant code snippet from `NodeManager` is:

```solidity
/// @notice Checks if the contract supports an interface.
/// @param interfaceId The ID of the interface to check.
/// @return A boolean indicating whether the contract supports the interface.
function supportsInterface(bytes4 interfaceId) external pure override returns (bool) {
    return interfaceId == type(INodeManager).interfaceId;
}
```

## Impact Details

The incorrect implementation of `supportsInterface` can lead to major compatibility issues. Contracts and systems that check for EIP-165 compliance may fail to interact with `NodeManager` correctly. This could cause failures in contract interactions, integrations, and potentially lead to the malfunction of decentralized applications relying on `NodeManager`.

## Recommendation:

Update the `supportsInterface` function as follows:

```diff
/// @notice Checks if the contract supports an interface.
/// @param interfaceId The ID of the interface to check.
/// @return A boolean indicating whether the contract supports the interface.
function supportsInterface(bytes4 interfaceId) external pure override returns (bool) {
-    return interfaceId == type(INodeManager).interfaceId;
+    return interfaceId == this.supportsInterface.selector;
}
```

## References

* [NodeManager.supportsInterface() implementation](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/modules/NodeManager.sol#L51)
* [EIP-165 Standard](https://eips.ethereum.org/EIPS/eip-165)

## Proof of concept

## Proof of Concept

Add the following code to the test file `test/oracle/modules/NodeManager.test.ts`:

```javascript
describe("Chista0x-NodeManager", () => {
  it("Should successfully supportInterface with 0x01ffc9a7", async () => {
    const state = await nodeManager.supportsInterface("0x01ffc9a7");
    expect(state).to.be.equal(true);
  });

  it("Should successfully supportInterface with 0xffffffff", async () => {
    const state = await nodeManager.supportsInterface("0xffffffff");
    expect(state).to.be.equal(false);
  });

  it("Should successfully supportInterface with 0x12345678", async () => {
    const state = await nodeManager.supportsInterface("0x12345678");
    expect(state).to.be.equal(false);
  });  
});
```

Run the test with the command `npx hardhat test --grep "Chista0x-NodeManager"`

Test output:

```
  NodeManager
    Chista0x-NodeManager
      1) Should successfully supportInterface with 0x01ffc9a7
      ✔ Should successfully supportInterface with 0xffffffff
      ✔ Should successfully supportInterface with 0x12345678

  2 passing (3s)
  1 failing

  1) NodeManager
       Chista0x-NodeManager
         Should successfully supportInterface with 0x01ffc9a7:

      AssertionError: expected false to equal true
      + expected - actual

      -false
      +true

      at Context.<anonymous> (test\oracle\modules\NodeManager.test.ts:68:27)
```
