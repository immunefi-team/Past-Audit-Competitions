# 25885 - \[SC - Insight] Prevent the operator from submitting blocks to L

Submitted on Nov 20th 2023 at 22:28:23 UTC by @infosec\_us\_team for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25885

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Permanent freezing of funds in the Default Deposit Contract that is less than 2,500,000 USD.
* Force DeGate into Exodus Mode

## Description

The attack vector causes the following impact:

**1-** Operators will be unable to submit blocks to L1.

**2-** Funds from all new unconfirmed standard deposits (deposits where users directly transfer tokens to the Default Deposit Contract) will be frozen inside the Default Deposit Contract. Users can't withdraw those funds because operators can't confirm their deposit on L1.

**3-** As designed for this type of situation, an Exodus Mode will be forced, allowing previous users to withdraw their funds using a MerkleTree proof (by calling `withdrawFromMerkleTree(...)`). Unconfirmed standard deposits will stay stuck inside the DefaultDepositContract because new blocks can't be confirmed once the Exodus Mode begins.

**4-** The stake of the exchange can be burned by calling `burnExchangeStake()`.

## About the severity assessment

The severity is `High` under the following impacts:

* Force DeGate into Exodus Mode
* Permanent freezing of funds in the Default Deposit Contract that is less than 2,500,000 USD

## Background

DeGate supports External Owned Accounts (EOA) and Contract Accounts (CA). The only requirement is that the EVM account must have at least one transfer-in or transfer-out record.

> As a reference: <https://docs.degate.com/v/product\\_en/main-features/account-registration#account-registration-criteria>

There are two forms of signatures in DeGate: type == 0: EDDSA signature, verified in circuit type == 1: ECDSA signature, verified within the ExchangeV3 smart contract

> Reference: `/loopring_v3/circuit/Circuits/AccountUpdateCircuit.h` line #48

The DeGate protocol supports 3 types of ECDSA signature and verification methods:

1. Open signature (ETH\_Sign)
2. Structured signature (EIP-712)
3. Smart contract support (EIP-1271)

> Reference: <https://docs.degate.com/v/product\\_en/concepts/secret-key-and-signatures#ecdsa-signature-types>

The node verifies off-chain both the ECDSA signature and the EdDSA signature simultaneously. The circuit verifies only the EdDSA signature, while the smart contract only verifies the ECDSA signature.

## Bug description

The function `requireAuthorizedTx(from, signature, txHash);` from the `ExchangeSignatures.sol` library, imported in `ExchangeV3.sol`, is used in the **Withdraw** and **AccountUpdate** flow to verify that the signature is valid.

```javascript
function requireAuthorizedTx(
    ExchangeData.State storage S,
    address signer,
    bytes memory signature,
    bytes32 txHash
    )
    internal // inline call
{
    require(signer != address(0), "INVALID_SIGNER");
    // Verify the signature if one is provided, otherwise fall back to an approved tx
    if (signature.length > 0) {
        require(txHash.verifySignature(signer, signature), "INVALID_SIGNATURE");
    } else {
        require(S.approvedTx[signer][txHash], "TX_NOT_APPROVED");
        delete S.approvedTx[signer][txHash];
    }
}
```

In `txHash.verifySignature(...)` if the signer is a smart contract, the `verifyERC1271Signature(...)` function will be used to check the signature.

```javascript
function verifySignature(
    bytes32        signHash,
    address        signer,
    bytes   memory signature
    )
    internal
    view
    returns (bool)
{
    if (signer == address(0)) {
        return false;
    }

    return signer.isContract()?
        verifyERC1271Signature(signHash, signer, signature):
        verifyEOASignature(signHash, signer, signature);
}
```

The `verifyERC1271Signature(...)` function will make a staticcall to the smart contract and ask him if the signature is valid.

An evil EIP-1271 Contract Account can be created that makes the same signature pass or revert on convenience (for example, fail only on mainnet calls, but succeed on forks or off-chain tests).

Failing the signature verification check will bubble up to the `require(txHash.verifySignature(signer, signature), "INVALID_SIGNATURE");` statement inside the `requireAuthorizedTx(...)` function we previously shared, reverting the entire transaction.

Requests with an evil EIP-1271 Contract Account can be repeatedly submitted to the L2, creating a denial of service in L1.

Below, we'll share how to fully prevent this and other attack vectors we may have not though of yet, related to contract accounts and EIP-1271.

## Permanent fix

To safely support contract accounts, we have to modify the `requireAuthorizedTx(...)` function as follows:

**Step 1-** Verify if the `signer` param is a smart contract. **Step 2-** If it is a smart contract, skip signature verification and require for the transaction to be approved instead, using `require(S.approvedTx[signer][txHash], "TX_NOT_APPROVED");` **Step 3-** If is not a smart contract, proceed with the signature verification as usual.

This is safe, because once a smart contract has approved a tx in the `approvedTx[signer][txHash]`, he can't remove this approval until the transaction is executed by the node operator.

Permanently fixed function:

```javascript
// 0- Import AddressUtil lib
using AddressUtil for address;

function requireAuthorizedTx(
    ExchangeData.State storage S,
    address signer,
    bytes memory signature,
    bytes32 txHash
) internal {
    require(signer != address(0), "INVALID_SIGNER");

    // 1- Verify if the signer is a smart contract account
    if (signer.isContract()) {

        // 2- Require for the transaction to be approved
        require(S.approvedTx[signer][txHash], "TX_NOT_APPROVED");
        delete S.approvedTx[signer][txHash];

    // If is an EOA
    } else {

        // 3- Verify the signature if one is provided, otherwise fall back to an approved tx
        if (signature.length > 0) {
            require(txHash.verifySignature(signer, signature), "INVALID_SIGNATURE");
        } else {
            require(S.approvedTx[signer][txHash], "TX_NOT_APPROVED");
            delete S.approvedTx[signer][txHash];
        }
    }
}
```

## Proof of concept

As a proof of concept here's the code for an oversimplified evil EIP-1271 Contract Account that makes the same signature succeed or revert on convenience.

```solidity
contract Evil {

    bytes4 constant internal ERC1271_MAGICVALUE = 0x1626ba7e;

    bool internal active = true;

    // A function to simulate the ERC1271 `isValidSignature` function
    function isValidSignature(bytes32 _hash, bytes memory _signature) public view returns (bytes4 magicValue) {
        if (active) {
            return 0x20c13b0b;
        } else {
            return 0x00000000;
        }
    }
    // A function to modify `activate`
    function setActive(bool _value) public {
        active = _value;
    }
}
```

By default, all signatures will succeed. But you can front-run any specific transaction from the mempool of mainnet (like an operator's attempt to submit a block) and call `setActive(false)` to make the check for the signature conveniently fail.
