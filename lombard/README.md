# Lombard

## Reports by Severity

[Medium](<README.md#medium>) | [Low](<README.md#low>) | [Insight](<README.md#insight>)
<details>

<summary>Medium</summary>

* [38634 - [SC - Medium] Insufficient validation on offchainTokenData in TokenPool.releaseOrMint allows CCIP message to be executed with mismatched payload potentially leading to loss of funds in cross-ch...](./38634-sc-medium-insufficient-validation-on-offchaintokendata-in-tokenpool.releaseormint-allows-ccip.md)
* [38066 - [SC - Medium] \`ProxyFactory\` is vulnerable to DoS/Address Hijacking](./38066-sc-medium-proxyfactory-is-vulnerable-to-dos-address-hijacking.md)
* [38154 - [SC - Medium] The offchain data provided to the CLAdapter isn’t properly validated and can be from a different CCIP message, resulting in the freezing of funds](./38154-sc-medium-the-offchain-data-provided-to-the-cladapter-isnt-properly-validated-and-can-be-from.md)
* [38342 - [SC - Medium] Interchanging \`offchainTokenData\` between two valid messages](./38342-sc-medium-interchanging-offchaintokendata-between-two-valid-messages.md)
* [38363 - [SC - Medium] LBTC cross-chain transfer can be DOSed](./38363-sc-medium-lbtc-cross-chain-transfer-can-be-dosed.md)
* [38335 - [SC - Medium] Attacker can exploit PartnerVault mint small amount to cause LBTC depeg or Protocol Insolvency](./38335-sc-medium-attacker-can-exploit-partnervault-mint-small-amount-to-cause-lbtc-depeg-or-protocol.md)

</details>

<details>

<summary>Low</summary>

* [38344 - [SC - Low] Old validated messages can not pass proof check when new validators are set](./38344-sc-low-old-validated-messages-can-not-pass-proof-check-when-new-validators-are-set.md)
* [38137 - [SC - Low] \`RateLimits\` library incorrectly reset the consumed amount when the limit is updated](./38137-sc-low-ratelimits-library-incorrectly-reset-the-consumed-amount-when-the-limit-is-updated.md)
* [38231 - [SC - Low] Due to incorrect design in \`Consortium::setNextValidatorSet\` the validator set could not be set in certain valid scenarios](./38231-sc-low-due-to-incorrect-design-in-consortium-setnextvalidatorset-the-validator-set-could-not-b.md)
* [38286 - [SC - Low] BitcoinUtils.getDustLimitForOutput calculate wrongly the dust limit for a given Bitcoin script public key](./38286-sc-low-bitcoinutils-getdustlimitforoutput-calculate-wrongly-the-dust-limit-for-a-given-bitcoin.md)

</details>

<details>

<summary>Insight</summary>

* [38644 - [SC - Insight] Q\&A](./38644-sc-insight-q-and-a.md)
* [38102 - [SC - Insight] Due to incorrect design in \`BasculeV2::validateWithdrawal\` valid transactions will be reverted, which will make protocol unable to mint tokens](./38102-sc-insight-due-to-incorrect-design-in-basculev2-validatewithdrawal-valid-transactions-will-be.md)
* [38012 - [SC - Insight] Unused Function in CLAdapter Contract](./38012-sc-insight-unused-function-in-cladapter-contract.md)
* [38116 - [SC - Insight] Partner vaults don't account for FireBridge fees, forcing LBTC burn to never work](./38116-sc-insight-partner-vaults-dont-account-for-firebridge-fees-forcing-lbtc-burn-to-never-work.md)
* [38148 - [SC - Insight] Unnecessary Storage Pointer Declaration batchMintWithFee](./38148-sc-insight-unnecessary-storage-pointer-declaration-batchmintwithfee.md)
* [38189 - [SC - Insight] Attacker can grief calls to \`lbtc.mintWithFee()\`](./38189-sc-insight-attacker-can-grief-calls-to-lbtc.mintwithfee.md)
* [38225 - [SC - Insight] user funds will get stuck if \`removeDestination\` executes before notarization and withdraw.](./38225-sc-insight-user-funds-will-get-stuck-if-removedestination-executes-before-notarization-and-wit.md)
* [38257 - [SC - Insight] Freezing of msg.value passed in Bridge.deposit() if adapter is address zero](./38257-sc-insight-freezing-of-msg.value-passed-in-bridge.deposit-if-adapter-is-address-zero.md)
* [38341 - [SC - Insight] Suboptimal gas usage and ambiguous behavior during fee estimation](./38341-sc-insight-suboptimal-gas-usage-and-ambiguous-behavior-during-fee-estimation.md)
* [38370 - [SC - Insight] Issue Between Comment and Code in Consortium](./38370-sc-insight-issue-between-comment-and-code-in-consortium.md)

</details>

## Reports by Type

[Smart Contract](<README.md#smart-contract>)
<details>

<summary>Smart Contract</summary>

* [38644 - [SC - Insight] Q\&A](./38644-sc-insight-q-and-a.md)
* [38344 - [SC - Low] Old validated messages can not pass proof check when new validators are set](./38344-sc-low-old-validated-messages-can-not-pass-proof-check-when-new-validators-are-set.md)
* [38137 - [SC - Low] \`RateLimits\` library incorrectly reset the consumed amount when the limit is updated](./38137-sc-low-ratelimits-library-incorrectly-reset-the-consumed-amount-when-the-limit-is-updated.md)
* [38102 - [SC - Insight] Due to incorrect design in \`BasculeV2::validateWithdrawal\` valid transactions will be reverted, which will make protocol unable to mint tokens](./38102-sc-insight-due-to-incorrect-design-in-basculev2-validatewithdrawal-valid-transactions-will-be.md)
* [38634 - [SC - Medium] Insufficient validation on offchainTokenData in TokenPool.releaseOrMint allows CCIP message to be executed with mismatched payload potentially leading to loss of funds in cross-ch...](./38634-sc-medium-insufficient-validation-on-offchaintokendata-in-tokenpool.releaseormint-allows-ccip.md)
* [38012 - [SC - Insight] Unused Function in CLAdapter Contract](./38012-sc-insight-unused-function-in-cladapter-contract.md)
* [38066 - [SC - Medium] \`ProxyFactory\` is vulnerable to DoS/Address Hijacking](./38066-sc-medium-proxyfactory-is-vulnerable-to-dos-address-hijacking.md)
* [38116 - [SC - Insight] Partner vaults don't account for FireBridge fees, forcing LBTC burn to never work](./38116-sc-insight-partner-vaults-dont-account-for-firebridge-fees-forcing-lbtc-burn-to-never-work.md)
* [38148 - [SC - Insight] Unnecessary Storage Pointer Declaration batchMintWithFee](./38148-sc-insight-unnecessary-storage-pointer-declaration-batchmintwithfee.md)
* [38154 - [SC - Medium] The offchain data provided to the CLAdapter isn’t properly validated and can be from a different CCIP message, resulting in the freezing of funds](./38154-sc-medium-the-offchain-data-provided-to-the-cladapter-isnt-properly-validated-and-can-be-from.md)
* [38189 - [SC - Insight] Attacker can grief calls to \`lbtc.mintWithFee()\`](./38189-sc-insight-attacker-can-grief-calls-to-lbtc.mintwithfee.md)
* [38225 - [SC - Insight] user funds will get stuck if \`removeDestination\` executes before notarization and withdraw.](./38225-sc-insight-user-funds-will-get-stuck-if-removedestination-executes-before-notarization-and-wit.md)
* [38257 - [SC - Insight] Freezing of msg.value passed in Bridge.deposit() if adapter is address zero](./38257-sc-insight-freezing-of-msg.value-passed-in-bridge.deposit-if-adapter-is-address-zero.md)
* [38231 - [SC - Low] Due to incorrect design in \`Consortium::setNextValidatorSet\` the validator set could not be set in certain valid scenarios](./38231-sc-low-due-to-incorrect-design-in-consortium-setnextvalidatorset-the-validator-set-could-not-b.md)
* [38286 - [SC - Low] BitcoinUtils.getDustLimitForOutput calculate wrongly the dust limit for a given Bitcoin script public key](./38286-sc-low-bitcoinutils-getdustlimitforoutput-calculate-wrongly-the-dust-limit-for-a-given-bitcoin.md)
* [38341 - [SC - Insight] Suboptimal gas usage and ambiguous behavior during fee estimation](./38341-sc-insight-suboptimal-gas-usage-and-ambiguous-behavior-during-fee-estimation.md)
* [38342 - [SC - Medium] Interchanging \`offchainTokenData\` between two valid messages](./38342-sc-medium-interchanging-offchaintokendata-between-two-valid-messages.md)
* [38363 - [SC - Medium] LBTC cross-chain transfer can be DOSed](./38363-sc-medium-lbtc-cross-chain-transfer-can-be-dosed.md)
* [38370 - [SC - Insight] Issue Between Comment and Code in Consortium](./38370-sc-insight-issue-between-comment-and-code-in-consortium.md)
* [38335 - [SC - Medium] Attacker can exploit PartnerVault mint small amount to cause LBTC depeg or Protocol Insolvency](./38335-sc-medium-attacker-can-exploit-partnervault-mint-small-amount-to-cause-lbtc-depeg-or-protocol.md)

</details>
