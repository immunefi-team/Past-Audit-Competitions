# #46856 \[SC-Medium] The calculation of shares obtained through token trades will be incorrect, causing users to pay excessive yield fees.

**Submitted on Jun 5th 2025 at 11:20:46 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46856
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/vaults>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When users make deposits, the contract records the asset amount in order to calculate and collect yield fees upon withdrawal. However, since shares tokens can be obtained through token transfer transactions, and the contract does not track the deposited\_assets for users who acquire shares this way, it may result in users unexpectedly paying excessive yield fees.

## Vulnerability Details

```rust
        fn deposit(ref self: ContractState, assets: u256, receiver: ContractAddress) {
            ...
            // caller address is only used to pull funds from asset
            // rest of the logic is based on the receiver address
            erc20_disp.transferFrom(caller, this, assets);

            let current_balance = self.asset_balances.read(receiver);
            let new_balance = current_balance + assets;
            self.asset_balances.write(receiver, new_balance);
```

During the deposit process, the system records the amount of assets deposited by the user.

```rust
        fn request_withdrawal(ref self: ContractState, shares: u256) {
            ...
            // fraction of the total shares to withdraw
            let withdraw_fraction = (shares * CONSTANTS::WAD) / account_shares;
            // total deposited assets of the user
            let deposited_assets = self.asset_balances.read(caller);
            // assets to withdraw
            let withdraw_assets = (deposited_assets * shares) / account_shares;
            // value of the shares to withdraw
            let shares_value = self._convert_to_assets(shares);
            // assets after withdrawal
            let assets_after_withdrawal = deposited_assets - withdraw_assets;
            assert(assets_after_withdrawal >= 0, Errors::ZERO_AMOUNT);
            // not needed? if vault liquidated - this would prevent burning tokens.
            // shares value > USDC_TO_WAD because of asset decimals
            assert(shares_value > 0, Errors::ZERO_AMOUNT);
            ...
            // profit share in shares
            let mut profit_share_shares = 0;
            let profit_share_percentage: u256 = self.profit_share_percentage().into();
            // owner is not subject to profit share
            if (shares_value > withdraw_assets && profit_share_percentage > 0 && !is_owner) {
                // profit of the account
                let profit_assets = shares_value - withdraw_assets;
                // profit share in assets
                let profit_share_assets = (profit_assets * profit_share_percentage) / 100;
                // profit share in shares
                profit_share_shares = self
                    ._convert_to_shares(profit_share_assets, ROUND_STRATEGY_UP);
                // transfer profit share to the owner from vault (transferred to vault before)
                self.erc20._transfer(caller, owner, profit_share_shares);
            }
```

This value is used in request\_withdrawal to calculate share yields and pay yield fees to the owner.

However, sharesToken can be transferred without updating the asset\_balances mapping. When users acquire sharesToken through trading (without depositing), their asset\_balances remain unrecorded in the contract. This causes the yield fee calculation to incorrectly apply to both principal + yield (rather than just yield), forcing them to unexpectedly pay massive yield fees and suffer severe financial losses.

## Impact Details

The yield fee rate may be incorrectly applied directly to the principal amount. Users who acquire shares tokens through trading will consequently suffer severe financial losses.

This creates a backdoor for initial depositors, allowing them to sell the shares representing their yield portion on the market and shift their yield fee obligations to unsuspecting buyers.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L251> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L525> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L636>

## Proof of Concept

## Proof of Concept

Consider the following scenario:

1. User1 is not a vault deposit participant but acquires 10,000 shares Token from the market when asset/shares = 1. Their asset\_balances in the contract remains 0.
2. After some time, the vault generates yield, making asset/share = 1.1.
3. When the user withdraws 10,000 shares:
   * With 20% yield fee rate
   * shares\_value = 10,000 \* 1.1 = 11,000
   * Since asset\_balances = 0, withdraw\_assets = 0
   * profit\_assets = 11,000 - 0 = 11,000 (incorrect calculation)
   * profit\_share\_assets = 11,000 \* 20% = 2,200
   * profit\_share\_shares = 2,000 (shares to be paid as fee)

Actual correct calculation:

* During their holding period, the user only earned 1,000 asset in profit
* Proper fee should be: 1,000 \* 20% / 1.1 = 181 shares
* Result: User overpays by more than 1,800 shares in fees
