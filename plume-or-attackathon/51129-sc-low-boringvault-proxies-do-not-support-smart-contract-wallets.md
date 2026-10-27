# 51129 sc low boringvault proxies do not support smart contract wallets

**Submitted on Jul 31st 2025 at 12:18:30 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51129
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Vulnerability Details

The `TellerWithMultiAssetSupportPredicateProxy` uses the `PredicateClient` to verify signatures.

```solidity
    function deposit(
        ERC20 depositAsset,
        uint256 depositAmount,
        uint256 minimumMint,
        address recipient,
        CrossChainTellerBase teller,
        PredicateMessage calldata predicateMessage
    )
        external
        nonReentrant
        returns (uint256 shares)
    {
        if (paused()) {
            revert TellerWithMultiAssetSupportPredicateProxy__Paused();
        }

        bytes memory encodedSigAndArgs = abi.encodeWithSignature("deposit()");
        // @audit-issue the signature method is incompatible with smart wallets that use eip-1271.
@>        if (!_authorizeTransaction(predicateMessage, encodedSigAndArgs, msg.sender, 0)) {
            revert TellerWithMultiAssetSupportPredicateProxy__PredicateUnauthorizedTransaction();
        }
... 
```

The problem is `_authorizeTransaction` does not support smart contract wallets:

<https://github.com/predicatelabs/predicate-contracts/blob/32d16b289752951c6dfe51b7efd2ec1c17b63e5c/src/ServiceManager.sol#L335>

```solidity
    function validateSignatures(
        Task calldata _task,
        address[] memory signerAddresses,
        bytes[] memory signatures
    ) external returns (bool isVerified) {
      ...
@>            address recoveredSigner = ECDSA.recover(messageHash, signatures[i]);
```

This will lead the transaction to revert because, with EIP-7702, it is now possible for addresses to have `code.length > 0` but still use their private keys to sign;

### Impact Details

{% hint style="warning" %}
Smart contract wallets cannot interact with the `TellerWithMultiAssetSupportPredicateProxy` as `ECDSA.recover` from `_authorizeTransaction` will cause the transaction to revert.
{% endhint %}

### Recommendation

{% hint style="info" %}
The use of `SignatureChecker` would resolve this issue, as it supports EOA and ERC‑1271 signatures. For example: `SignatureChecker.isValidERC1271SignatureNow`

OpenZeppelin docs: <https://docs.openzeppelin.com/contracts/5.x/api/utils/cryptography#SignatureChecker>
{% endhint %}

## References

<details>

<summary>Relevant links</summary>

* EIP7702 - <https://eip7702.io/>
* OZ `recover` function: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol#L115-L119>
* Predicate ServiceManager (validateSignatures): <https://github.com/predicatelabs/predicate-contracts/blob/32d16b289752951c6dfe51b7efd2ec1c17b63e5c/src/ServiceManager.sol#L335>

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Smart contract wallet attempts deposit

User is a Smart Contract Wallet that wants to deposit funds through `TellerWithMultiAssetSupportPredicateProxy`.
{% endstep %}

{% step %}

### User signs transaction using smart contract wallet address

User signs the transaction with their private key, but uses the smart contract wallet address and calls the `deposit` function.
{% endstep %}

{% step %}

### Predicate constructs hash including smart contract wallet address

The hash built in `PredicateClient.ServiceManager.validateSignatures` utilises the smart contract wallet address to compare the hash against the signature:

<https://github.com/predicatelabs/predicate-contracts/blob/32d16b289752951c6dfe51b7efd2ec1c17b63e5c/src/ServiceManager.sol#L335>

```solidity
@> bytes32 messageHash = hashTaskSafe(_task);
...
@>            address recoveredSigner = ECDSA.recover(messageHash, signatures[i]);

function hashTaskSafe(
        Task calldata _task
    ) public view returns (bytes32) {
        return keccak256(
            abi.encode(
                _task.taskId,
@>                _task.msgSender, // @audit smart contract wallet address
                msg.sender,
                _task.value,
                _task.encodedSigAndArgs,
                _task.policyID,
                _task.quorumThresholdCount,
                _task.expireByTime
            )
        );
    }
```

{% endstep %}

{% step %}

### Transaction reverts

`ECDSA.recover` fails to validate and the transaction reverts. The user cannot deposit into the vault using `TellerWithMultiAssetSupportPredicateProxy`.
{% endstep %}
{% endstepper %}
