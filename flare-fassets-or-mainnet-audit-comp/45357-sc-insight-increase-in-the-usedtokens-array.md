# #45357 \[SC-Insight] Increase in the usedTokens array

**Submitted on May 13th 2025 at 09:54:50 UTC by @vargalove for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45357
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**

## Description

## Brief/Intro

In the AgentVault contract, the private function \_tokenUsed(IERC20 \_token, uint256 \_flags) is the only one that adds elements to the usedTokens array, and there is no function that removes tokens from usedTokens, not even in destroy(), where they are only iterated to transfer balances and revoke delegations.

## Vulnerability Details

This means that once a token is used, it will always remain recorded in usedTokens, even if its balance is zero or the agent was destroyed in the next function:

function destroy(address payable \_recipient) external override onlyAssetManager nonReentrant { uint256 length = usedTokens.length; for (uint256 i = 0; i < length; i++) { IERC20 token = usedTokens\[i]; uint256 useFlags = tokenUseFlags\[token]; // undelegate all governance delegation if ((useFlags & TOKEN\_DELEGATE\_GOVERNANCE) != 0) { IWNat(address(token)).governanceVotePower().undelegate(); } // undelegate all WNat delegation if ((useFlags & TOKEN\_DELEGATE) != 0) { IVPToken(address(token)).undelegateAll(); } // transfer balance to recipient if ((useFlags & TOKEN\_DEPOSIT) != 0) { uint256 balance = token.balanceOf(address(this)); if (balance > 0) { token.safeTransfer(\_recipient, balance); } } } // transfer native balance, if any (used to be done by selfdestruct) Transfers.transferNAT(\_recipient, address(this).balance); }

## Impact Details

The impact is that the array can grow indefinitely over time as tokens are added to the contract. This can increase the gas price in the contract each time onlyAssetManager calls the destroy(address payable \_recipient) function. It is known that when the destroy function is called, it has to search for the usedTokens in the length of the array:

uint256 length = usedTokens.length; for (uint256 i = 0; i < length; i++) { IERC20 token = usedTokens\[i]; uint256 useFlags = tokenUseFlags\[token];

If there are many tokens in that array, the gas price increases.

## References

<https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>

## Proof of Concept

## Proof of Concept

1. Tokens begin to be added in the following function depositNat(IWNat \_wNat) external payable override onlyAssetManager { \_wNat.deposit{value: msg.value}(); assetManager.updateCollateral(address(this), \_wNat); \_tokenUsed(\_wNat, TOKEN\_DEPOSIT); }
2. This causes the usedTokens array to increment because it calls the following private function:

function \_tokenUsed(IERC20 \_token, uint256 \_flags) private { if (tokenUseFlags\[\_token] == 0) { usedTokens.push(\_token); } tokenUseFlags\[\_token] |= \_flags; }

Pushing the IERC20\[] array private usedTokens; adding a new token.

3. Over time, if tokens continue to be added to the array, the gas cost will be much higher, since there is no function to remove the usedTokens from the array.
4. To mitigate this, ideally for code optimization, you should add a function that removes the usedTokens from the array if, for example: It has a balance of 0, it is not used in the agent contract, etc.

function removeUsedToken(IERC20 \_token) external onlyOwner { uint256 length = usedTokens.length; for (uint256 i = 0; i < length; i++) { if (usedTokens\[i] == \_token) { // Swap and pop usedTokens\[i] = usedTokens\[length - 1]; usedTokens.pop();

```
        tokenUseFlags[_token] = 0;
        return;
    }
}
revert("Token not found in usedTokens");
```

}

This is to demonstrate that it is a function that can be added to optimize the code in more extreme cases, and that it costs nothing to the protocol to add it.
