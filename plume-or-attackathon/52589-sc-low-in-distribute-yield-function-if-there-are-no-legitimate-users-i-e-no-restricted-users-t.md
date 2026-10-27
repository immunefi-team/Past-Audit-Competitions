# 52589 sc low in distribute yield function if there are no legitimate users i e no restricted users the funds will remain stuck

* Submitted on Aug 11th 2025 at 19:49:15 UTC by @swarun for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #52589
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

## Impacts

* Temporary freezing of funds for at least 1 hour
* Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

In the `distributeYield` function the contract first transfers the tokens into the ArcToken contract and only afterwards checks if there are any legitimate (unrestricted) users. If none exist, the function returns after the transfer and the funds remain stuck in the contract.

### Vulnerability Details

`distributeYield` performs the incoming token transfer first (via `safeTransferFrom`) and then checks whether there are any holders eligible to receive yield. If there are no eligible holders (effective supply is zero), the function emits `YieldDistributed(0, yieldTokenAddr)` and returns, leaving the transferred tokens held by the contract with no mechanism to extract them.

### Impact Details

Yield tokens can become permanently or temporarily stuck in the contract, effectively lost to users.

## Proof of Concept

<details>

<summary>distributeYield function (excerpt)</summary>

```solidity
function distributeYield(
        uint256 amount
    ) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
        ArcTokenStorage storage $ = _getArcTokenStorage();

        if (amount == 0) {
            revert ZeroAmount();
        }

        uint256 supply = totalSupply();
        if (supply == 0) {
            revert NoTokensInCirculation();
        }

        address yieldTokenAddr = $.yieldToken;
        if (yieldTokenAddr == address(0)) {
            revert YieldTokenNotSet();
        }

        ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
        yToken.safeTransferFrom(msg.sender, address(this), amount);

        uint256 distributedSum = 0;
        uint256 holderCount = $.holders.length();
        if (holderCount == 0) {
            emit YieldDistributed(0, yieldTokenAddr);
            return;
        }

        uint256 effectiveTotalSupply = 0;
        for (uint256 i = 0; i < holderCount; i++) {
            address holder = $.holders.at(i);
            if (_isYieldAllowed(holder)) {
                effectiveTotalSupply += balanceOf(holder);
            }
        }

        if (effectiveTotalSupply == 0) {
            emit YieldDistributed(0, yieldTokenAddr);
            return;
        }

        uint256 lastProcessedIndex = holderCount > 0 ? holderCount - 1 : 0;
        for (uint256 i = 0; i < lastProcessedIndex; i++) {
            address holder = $.holders.at(i);

            if (!_isYieldAllowed(holder)) {
                continue;
            }

            uint256 holderBalance = balanceOf(holder);
            if (holderBalance > 0) {
                uint256 share = (amount * holderBalance) / effectiveTotalSupply;
                if (share > 0) {
                    yToken.safeTransfer(holder, share);
                    distributedSum += share;
                }
            }
        }

        if (holderCount > 0) {
            address lastHolder = $.holders.at(lastProcessedIndex);
            if (_isYieldAllowed(lastHolder)) {
                uint256 lastShare = amount - distributedSum;
                if (lastShare > 0) {
                    yToken.safeTransfer(lastHolder, lastShare);
                    distributedSum += lastShare;
                }
            }
        }

        emit YieldDistributed(distributedSum, yieldTokenAddr);
    }
```

As shown above, the transfer into the contract happens before checking `effectiveTotalSupply`. If `effectiveTotalSupply == 0` the function returns and the tokens remain in the contract.

The related function `distributionYieldWithLimit` applies the effective supply check before transferring tokens (i.e., it avoids this issue):

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L518>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L527>

</details>

## References

* Vulnerable function location: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L408>
* Related correct implementation: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L518>
