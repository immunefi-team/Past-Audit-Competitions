# 51502 sc low enabling transfer restrictions permanently blocks minting and burning

**Submitted on Aug 3rd 2025 at 12:55:14 UTC by @rajaroy43 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51502
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The `WhitelistRestrictions.sol` contract incorrectly implements its transfer validation logic by failing to account for the ERC20 standard's use of the zero address (`address(0)`) for minting and burning operations. When transfer restrictions are enabled, this flaw makes it impossible to mint new tokens or burn existing ones. In a production environment, this would permanently disable critical protocol functionalities, break the token's economic model, and halt any operations that depend on changing the total supply, such as token sales, rewards distribution, or deflationary mechanisms.

### Vulnerability Details

The vulnerability exists within the `isTransferAllowed` function of the `WhitelistRestrictions.sol` contract. When transfer restrictions are active (`transfersAllowed == false`), the function requires both the `from` and `to` addresses to be on the whitelist for a transfer to be approved.

The logic is as follows:

{% code title="WhitelistRestrictions.sol (relevant snippet)" %}

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/ ) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();

    // If transfers are unrestricted, allow all transfers
    if (ws.transfersAllowed) {
        return true;
    }

    // Otherwise, only allow if both the sender and receiver are whitelisted
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

{% endcode %}

<https://github.com/plumenetwork/contracts/blob/main/arc/src/restrictions/WhitelistRestrictions.sol#L110>

However, the ERC20 standard defines special cases for the zero address:

* Minting: a transfer from `address(0)`.
* Burning: a transfer to `address(0)`.

The `WhitelistRestrictions` contract correctly prevents `address(0)` from being added to the whitelist, meaning `ws.isWhitelisted[address(0)]` will always be `false`.

When an associated token contract (like `ArcToken`) attempts to `_mint` or `_burn`, its `_update` hook calls this `isTransferAllowed` function.

* For a mint, `isTransferAllowed(address(0), ...)` is called → evaluates to `false`.
* For a burn, `isTransferAllowed(..., address(0))` is called → evaluates to `false`.

Consequently, the token's `_update` function reverts the transaction, effectively blocking all minting and burning as long as the whitelist restriction is active.

### Impact Details

Permanent Functional Denial of Service: The `mint()` and `burn()` functions become completely and permanently unusable whenever the whitelist feature is activated. There is no way to perform these actions without disabling the restrictions entirely, defeating the purpose of the module.

### References

* <https://github.com/plumenetwork/contracts/blob/main/arc/src/ArcToken.sol#L671>
* <https://github.com/plumenetwork/contracts/blob/main/arc/src/restrictions/WhitelistRestrictions.sol#L110>

## Proof of Concept

{% stepper %}
{% step %}

### Enable transfer restrictions

Admin enables transfer restrictions:

```solidity
vm.prank(admin);
vulnerableRestrictions.setTransfersAllowed(false);
```

{% endstep %}

{% step %}

### Expect revert on mint

We expect the mint call to revert because `from` is `address(0)`:

```solidity
vm.expectRevert("TransferRestricted()");
```

{% endstep %}

{% step %}

### Attempt to mint

Attempt to mint, which triggers the flawed `isTransferAllowed` check:

```solidity
token.mint(userA, 100 ether);
```

{% endstep %}
{% endstepper %}
