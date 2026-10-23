# 51320 sc low malicious teller parameter allow event data manipulation

Submitted on Aug 1st 2025 at 16:55:10 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51320
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* Impacts: Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract accepts user-controlled `teller` parameters in both `deposit()` and `depositAndBridge()` functions. A malicious user can supply a fake teller that appears to perform deposit-related actions (depositing funds into the vault, minting shares, returning a nonce) but executes different logic. This enables emitting misleading events that off-chain systems might rely upon.

### Vulnerability Details

The `TellerWithMultiAssetSupportPredicateProxy` accepts the `teller` parameter as an input, meaning a user can pass any contract that conforms to the expected interface but implements arbitrary logic:

<https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L121-L170>

```solidity
    function depositAndBridge(
        ERC20 depositAsset,
        uint256 depositAmount,
        uint256 minimumMint,
        BridgeData calldata data,
@>        CrossChainTellerBase teller, // @audit user-controlled
        PredicateMessage calldata predicateMessage
    )
        external
        payable
        nonReentrant
    {
      ...
        }
        lastSender = msg.sender;
@>        ERC20 vault = ERC20(teller.vault());
        //approve vault to take assets from proxy
@>        depositAsset.safeApprove(address(vault), depositAmount);
        //transfer deposit assets from sender to this contract
        depositAsset.safeTransferFrom(msg.sender, address(this), depositAmount);
        // mint shares
@>        teller.depositAndBridge{ value: msg.value }(depositAsset, depositAmount, minimumMint, data);
        lastSender = address(0);
@>        uint96 nonce = teller.depositNonce();
        //get the current share lock period
        uint64 currentShareLockPeriod = teller.shareLockPeriod();
@>        AccountantWithRateProviders accountant = AccountantWithRateProviders(teller.accountant());
        //get the share amount
@>        uint256 shares = depositAmount.mulDivDown(10 ** vault.decimals(), accountant.getRateInQuoteSafe(depositAsset));

      // @audit - event data can be manipulated
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

Because off-chain systems often rely on emitted events to process actions (for example, bridging flows), a user-controlled `teller` can emit misleading events or cause the proxy to emit events whose data is not representative of the real state of the vault or bridge.

Currently, a more severe exploit (such as stealing funds) is mitigated by the auth system restricting calls to the base contract to only the proxy. The issue here is primarily about integrity of event data.

{% hint style="warning" %}
User-controlled `teller` input allows malicious contracts to emit fake Deposit events, misleading off-chain systems and faking cross-chain deposits.
{% endhint %}

## Recommendation

{% hint style="info" %}
Remove the `teller` parameter from the deposit functions and instead store the teller address in a state variable. Use this stored value internally within the deposit logic so callers cannot influence which teller implementation is used.
{% endhint %}

## Proof of Concept

Context: a user deploys a fake `teller` contract which returns manipulated values for nonce, lock period, and vault address and/or emits misleading events.

{% stepper %}
{% step %}

### Step

Malicious user deploys a fake `teller` contract that implements the expected interface but returns manipulated values (nonce, share lock period, vault address) and/or emits misleading events.
{% endstep %}

{% step %}

### Step

User calls the `depositAndBridge` function on the proxy, passing the malicious `teller` contract as the `teller` parameter along with legitimate deposit details.
{% endstep %}

{% step %}

### Step

The proxy calls into the provided `teller` (which may not have performed the real deposit, may have minted no shares, and may return falsified values), then the proxy emits a `Deposit` event based on the values it reads from that `teller`.
{% endstep %}

{% step %}

### Step

Off-chain systems ingest the emitted `Deposit` event and act on the corrupted data (nonce, shares, lock period, vault address), leading to false reporting or incorrect downstream actions (e.g., treating a deposit as bridged when it was not).
{% endstep %}
{% endstepper %}

No on-chain value is directly stolen in this scenario (per current auth restrictions), but event integrity is compromised and off-chain processes can be misled.
