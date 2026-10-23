# #41689 \[SC-Insight] Blacklisting a Kodiak vault unintentionally whitelists a previously blacklisted token

**Submitted on Mar 17th 2025 at 15:32:22 UTC by @peppef for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41689
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

In the `Zapper.sol` contract the methods `updateSwappableTokens()` and `updateWhitelistedKodiakVault()` helps the blacklisting token and vault processes.\
\`\`\`solidity

````
function updateSwappableTokens(address token, bool isWhitelisted) external override onlyOwner { 
    _updateWhitelistedTokens(token, isWhitelisted);
}

function _updateWhitelistedTokens(address token, bool isWhitelisted) private {  
    require(token != address(0), "Zapper: token is zero address");
    whitelistedTokens[token] = isWhitelisted;
    emit TokenWhitelisted(token, isWhitelisted);
}

function updateWhitelistedKodiakVault(address vault, bool isEnabled) external override onlyOwner {
    ... // other code
    whitelistedKodiakVaults[vault] = isEnabled;
    ... // other code
    if (!whitelistedTokens[token0]) {
        _updateWhitelistedTokens(token0, true);
    }
    if (!whitelistedTokens[token1]) {
        _updateWhitelistedTokens(token1, true);
    }
}
```
````

However they both edit the state of the mapping `_updateWhitelistedTokens[]` which can lead to a weird scenario where blacklisting a vault which involves a blacklisted token has the contrary effect to whitelist it again in `_updateWhitelistedTokens[]`.

## Proof of Concept

Suppose the following scenario:

* A token named $BAD\_TOKEN is blacklisted by the owner using the following code

  ```solidity
  updateSwappableTokens(bad_token_address, false)
  ```

  This, through `_updateWhitelistedTokens()` private function sets `whitelistedTokens[bad_token_address] = false`
* At this point zapIn and zapOut methods are still usable on vaults involving $BAD\_TOKEN since the `onlyWhitelistedKodiakVaults()` modifier only checks for whitelisted vaults in the mapping `whitelistedKodiakVaults[]`
* For this reason the owner proceeds to blacklist the first vault involving $BAD\_TOKEN, for example `wBERA/BAD_TOKEN` vault with

  ```solidity
  updateWhitelistedKodiakVault(wBera_BadToken_vault, false)
  ```

  This function call sets `whitelistedKodiakVaults[wBera_BadToken_vault] = false` but at the same time whitelists again bad\_token\_address inside `whitelistedTokens[]` which was previously blacklisted due to the how `updateWhitelistedKodiakVault()` is implemented
* Additionally the owner needs some time to blacklist all the other vaults involving $BAD\_TOKEN for example: HONEY/BAD\_TOKEN, USDT/BAD\_TOKEN, USDC/BAD\_TOKEN and thus, in the meanwhile, those vaults are still being usable by users and every potential blacklisting call to `updateSwappableTokens()` is useless due to next `updateWhitelistedKodiakVault()`call.
