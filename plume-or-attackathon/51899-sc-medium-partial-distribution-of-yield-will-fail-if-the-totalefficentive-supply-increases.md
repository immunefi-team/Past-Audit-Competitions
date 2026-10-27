# 51899 sc medium partial distribution of yield will fail if the totalefficentive supply increases&#x20;

* **Submitted on:** Aug 6th 2025 at 14:06:14 UTC by @TeamJosh for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51899
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

When distributing yields, the total effective supply is used to determine the share that each person gets based on their balance. This supply is calculated every time `distributeYieldWithLimit` is called. The total effective supply increases when more tokens are minted, and since the Arc token is mintable, there is a chance that this will occur.

### Vulnerability Details

Assuming that there are 10 holders, all holding 100 tokens each.

The admin distributes 100 yield to the first 5 token holders; they will each get 10 yield tokens. The next step will be to distribute the remaining 50 tokens to the remaining 5 token holders. This will work perfectly on a normal day; however, if the token total supply increases before the second batch is distributed, then the distribution will fail due to insufficient balance in the contract, and the remaining reward per holder will be diluted.

Relevant snippet:

```solidity
function distributeYieldWithLimit(//@audit users can frontrun to earn twice
        uint256 totalAmount,
        uint256 startIndex,
        uint256 maxHolders
    )
        external
        onlyRole(YIELD_DISTRIBUTOR_ROLE)
        nonReentrant
        returns (uint256 nextIndex, uint256 totalHolders, uint256 amountDistributed)
    {
        ArcTokenStorage storage $ = _getArcTokenStorage();
        address yieldTokenAddr = $.yieldToken;
        if (yieldTokenAddr == address(0)) {
            revert YieldTokenNotSet();
        }

        if (totalAmount == 0) {
            revert ZeroAmount();
        }

        uint256 supply = totalSupply();
        if (supply == 0) {
            revert NoTokensInCirculation();
        }

        totalHolders = $.holders.length();
        if (totalHolders == 0) {
            return (0, 0, 0);
        }

        if (startIndex >= totalHolders) {
            startIndex = 0;
        }

        uint256 endIndex = startIndex + maxHolders;
        if (endIndex > totalHolders) {
            endIndex = totalHolders;
        }

        uint256 batchSize = endIndex - startIndex;
        if (batchSize == 0) {
            return (0, totalHolders, 0);
        }

        uint256 effectiveTotalSupply = 0;
        for (uint256 i = 0; i < totalHolders; i++) { 
            address holder = $.holders.at(i);
            if (_isYieldAllowed(holder)) {
                effectiveTotalSupply += balanceOf(holder);
            }
        }

        if (effectiveTotalSupply == 0) {
            nextIndex = endIndex < totalHolders ? endIndex : 0;
            return (nextIndex, totalHolders, 0);
        }

        ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
        amountDistributed = 0;

        if (startIndex == 0) {
            yToken.safeTransferFrom(msg.sender, address(this), totalAmount);
            console.log("Total Amount: ", totalAmount);
        }

        for (uint256 i = 0; i < batchSize; i++) {
            uint256 holderIndex = startIndex + i;
            address holder = $.holders.at(holderIndex);

            if (!_isYieldAllowed(holder)) {
                continue;
            }

            uint256 holderBalance = balanceOf(holder);
            if (holderBalance > 0) {
    @->            uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
                if (share > 0) {
                    yToken.safeTransfer(holder, share);
                    amountDistributed += share;
                }
            }
        }

        nextIndex = endIndex < totalHolders ? endIndex : 0;

        //console.log("Amount Distributed: ", amountDistributed);

        if (nextIndex == 0) {
            emit YieldDistributed(totalAmount, yieldTokenAddr);
        }

        return (nextIndex, totalHolders, amountDistributed);
    }
```

Note: The Arc token admin/minter can be anyone, including a smart contract that controls the minting of tokens; they are not necessarily protocol admins.

## Impact Details

1. Rewards will be temporarily stuck as the `distributeYieldWithLimit` function will fail on subsequent batches.
2. Rewards for subsequent batches will be diluted.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol?utm\\_source=immunefi#L540>

## Proof of Concept

<details>

<summary>PoC test (click to expand)</summary>

Add the following test to the `test/ArcToken.t.sol` file.

```solidity
function test_POC2() public {

    for (uint160 i = 2; i <= 10; i++) {
        token.transfer(address(i), 100e18); // Transfer 0.1 ARC to each address
    }

    // Approve and distribute yield
    yieldToken.approve(address(token), YIELD_AMOUNT);

    token.distributeYieldWithLimit(YIELD_AMOUNT, 0, 5);

    // Mint to address 6
    token.mint(address(6), 100e18);

    token.distributeYieldWithLimit(YIELD_AMOUNT, 5, 10);
}
```

Output:

```bash
Encountered 1 failing test in test/ArcToken.t.sol:ArcTokenTest
[FAIL: ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 45454545454545454549 [4.545e19], 90909090909090909090 [9.09e19])] test_POC2() (gas: 1192212)
```

</details>
