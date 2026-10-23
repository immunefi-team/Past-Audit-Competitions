# 51746 sc low depositandbridge function of tellerwithmultiassetsupportpredicateproxy sol can not be paused

* Report ID: #51746
* Report Type: Smart Contract
* Severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* Impacts: Temporary freezing of funds for at least 1 hour

## Description

### Brief / Intro

The `TellerWithMultiAssetSupportPredicateProxy.sol` contract inherits from `Pausable.sol` and checks the paused state in `deposit(...)` and `depositAndBridge(...)`. However, `pause(...)` and `unpause(...)` functions are not implemented, so there is no way to pause the contract when a security vulnerability is detected.

### Vulnerability Details

The `deposit(...)` and `depositAndBridge(...)` functions check for pause and revert when paused, but there is no exposed function to set the contract into the paused state.

Relevant snippet:

```solidity
File: TellerWithMultiAssetSupportPredicateProxy.sol
function depositAndBridge(
        ERC20 depositAsset,
        uint256 depositAmount,
        uint256 minimumMint,
        BridgeData calldata data,//msgGas not validated with msg.value.
        CrossChainTellerBase teller,
        PredicateMessage calldata predicateMessage
    )
        external
        payable
        nonReentrant
    {
        if (paused()) {//@audit pause is checked but pause() function not implemented
            revert TellerWithMultiAssetSupportPredicateProxy__Paused();
        }

        //@dev This is NOT the actual function that is called, it is the against which the predicate is authorized
        bytes memory encodedSigAndArgs = abi.encodeWithSignature("depositAndBridge()");
        //still use 0 for msg.value since we only need validation against sender address
        if (!_authorizeTransaction(predicateMessage, encodedSigAndArgs, msg.sender, 0)) {
            revert TellerWithMultiAssetSupportPredicateProxy__PredicateUnauthorizedTransaction();
        }
        lastSender = msg.sender;
        ERC20 vault = ERC20(teller.vault());
        //approve vault to take assets from proxy
        depositAsset.safeApprove(address(vault), depositAmount);
        //transfer deposit assets from sender to this contract
        depositAsset.safeTransferFrom(msg.sender, address(this), depositAmount);
        // mint shares
        teller.depositAndBridge{ value: msg.value }(depositAsset, depositAmount, minimumMint, data);
        lastSender = address(0);
        uint96 nonce = teller.depositNonce();
        //get the current share lock period
        uint64 currentShareLockPeriod = teller.shareLockPeriod();
        AccountantWithRateProviders accountant = AccountantWithRateProviders(teller.accountant());
        //get the share amount
        uint256 shares = depositAmount.mulDivDown(10 ** vault.decimals(), accountant.getRateInQuoteSafe(depositAsset));

        emit Deposit(
            address(teller),
            data.destinationChainReceiver,
            address(depositAsset),
            depositAmount,
            shares,
            block.timestamp,
            currentShareLockPeriod,
            nonce > 0 ? nonce - 1 : 0,
            address(vault)
        );
    }
```

## Impact Details

There is no way to pause `TellerWithMultiAssetSupportPredicateProxy.sol` when a potential vulnerability is suspected, preventing an emergency freeze and thereby potentially allowing exploitation until other mitigations (e.g., timelocks, external controls) take effect.

## Recommendation

Consider implementing `pause()` and `unpause()` as owner-restricted wrappers around the inherited `_pause()` and `_unpause()` functions.

Suggested implementation:

```solidity
File: TellerWithMultiAssetSupportPredicateProxy.sol
function pause() public onlyOwner {
    _pause();
}

function unpause() public onlyOwner {
    _unpause();
}
```

{% hint style="info" %}
Ensure the owner access control is correct and that pausing these proxy functions does not break any necessary on-chain flows. If ownership is multisig or governed, confirm the governance process for emergency pauses.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}
A potential vulnerability is detected on `TellerWithMultiAssetSupportPredicateProxy.sol`.
{% endstep %}

{% step %}
The owner attempts to pause the contract, but `pause` and `unpause` functions are not implemented.
{% endstep %}

{% step %}
The contract cannot be paused, leaving the vulnerability exploitable until other mitigations are applied.
{% endstep %}
{% endstepper %}
