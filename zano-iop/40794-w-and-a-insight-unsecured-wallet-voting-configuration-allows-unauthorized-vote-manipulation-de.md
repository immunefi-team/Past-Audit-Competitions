# #40794 \[W\&A-Insight] Unsecured Wallet Voting Configuration Allows Unauthorized Vote Manipulation Des

**Submitted on Mar 4th 2025 at 00:15:44 UTC by @jovi for** [**IOP | Zano**](https://immunefi.com/audit-competition/iop-zano)

* **Report ID:** #40794
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/zano-iop/blob/main/src/wallet/wallets\\_manager.cpp>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.

## Description

**Summary**\
On a machine running multiple locally accessed wallets, an attacker without the wallet password can still change the wallet's voting configuration. This is unintended since each wallet is password-protected, and such changes should be impossible without the correct password.

***

## Vulnerability Details

**Location**\
In `wallets_manager.cpp`, the wallet software sets a single `votes_config_path` (for example, `"voting.json"`) that is not protected by any wallet password checks.

**Description**

* **Cause**: The wallet’s votes are stored outside the encrypted wallet file and loaded at runtime without further password enforcement.
* **Expected Behavior**: Changing a wallet’s votes should require its password.
* **Observed Behavior**: A local user with file-system access can edit this config file and cause wallet's they don't own the password to cast new or altered votes, even though they have never provided the wallet password.

**Code Snippet**

```cpp
void wallets_manager::init_wallet_entry(wallet_vs_options& wo, uint64_t id)
{
  ...
  // Sets an unprotected config path for votes
  wo.w->get()->set_votes_config_path(m_data_dir + "/" + CURRENCY_VOTING_CONFIG_DEFAULT_FILENAME);
  ...
}
```

No subsequent checks enforce the wallet’s password when reading or applying that file.

**Impact**\
An unauthorized individual can override or add votes in a wallet they do not own, compromising user autonomy in on-chain decision or governance processes.

## Proof of Concept

1. The attacker has local file-system access on a system hosting multiple wallets.
2. They locate the shared “voting.json” file (or its variant).
3. They modify it with their desired votes.
4. All the local wallets subsequently broadcast these changes in newly minted blocks or normal transactions—no password prompts occur.
