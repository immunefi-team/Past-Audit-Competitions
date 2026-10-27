# #41855 \[SC-Insight] User is able to circumvent blocklist check by utilizing Solidity's implementation

**Submitted on Mar 18th 2025 at 21:15:44 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41855
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement-layerzero-devtools/tree/main/examples/oft-evm-move-adapters/deploy-eth/src/MOVEOFTAdapter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Description

When examining `move's` OFTAdapter implementation, we observe blocklist checks in both the `debit_fungible_asset` and `credit` flows, as highlighted below:

**`debit_fungible_asset` Flow:**

```javascript
assert_not_blocklisted(sender);
```

**`credit` Flow:**

```javascript
// Deposit the extracted amount to the recipient, or redirect to the admin if the recipient is blocklisted
primary_fungible_store::transfer(
    escrow_signer,
    metadata(),
=>  redirect_to_admin_if_blocklisted(to, amount_ld),
    amount_ld
);
```

In all cases, these checks ensure that blocklisted users cannot receive or process funds.

However, when analyzing the Solidity implementation, we find no blocklist checks throughout the entire flow, including the in-scope sections. For instance, in `_debit`:

```javascript
function _debit(address _from, uint256 _amountLD, uint256 _minAmountLD, uint32 _dstEid)
    internal
    virtual
    override
    returns (uint256 amountSentLD, uint256 amountReceivedLD)
{
    _checkAndUpdateRateLimit(_dstEid, _amountLD);
    return super._debit(_from, _amountLD, _minAmountLD, _dstEid);
}
```

This function is invoked within the `send` flow, yet it lacks any blocklist verification.

### Impact

Without blocklist enforcement in the Solidity implementation, a blocklisted user can bypass restrictions enforced in the `.move` implementation simply by interacting with the Solidity contract.

As a result, a core security mechanism of the protocol—blocking users from sending or receiving bridge funds—is useless.

### Recommendation

Include blocklist checks inside the Solidity implementation, this way users are not able to circumvent this core invariant.

## Proof of Concept

### POC

* Bob wants to perform a cross-chain bridge call
* Bob is blocklisted however, and is therefore unable to call `debit_fungible_asset`
* Bob calls solidity's `send` flow which does not perform a blocklist check effectively circumventing this check.
* Bob is now able to perform the same bridge operation while being blocklisted due to the absence of this check.
