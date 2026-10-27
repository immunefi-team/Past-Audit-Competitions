# #42637 \[SC-Insight] When there is sufficient liquidity for executing reward distribution, token swapping should be skipped to avoid slippage loss

**Submitted on Mar 25th 2025 at 06:24:55 UTC by @h2134 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42637
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When there is sufficient liquidity for executing reward distribution, token swapping should be skipped to avoid slippage loss.

## Vulnerability Details

When admin distributes excess rewards in `StakeV2`, `executeRewardDistributionYeet()` or `executeRewardDistribution()` is called, both of the 2 functions call zapIn functions from `Zapper` to mint compounding vault shares through 3 steps:

1. Swapping tokens;
2. Depositing swapped tokens into Kodiak Vault to mint island tokens;
3. Depositing island tokens into compounding vault to mint vault shares.

The purpose of first step (swapping tokens) is to get enough liquidity so the second step (depositing into Kodiak Vault) can be fulfilled.

Kodiak Vault accepts `YEET` and `WBERA` tokens. When `executeRewardDistributionYeet()` is called, some of the `YEET` tokens in `StakeV2` is swapped to `WBERA` tokens, then both tokens are deposited into Kodiak Vault.

At the time of writing, `YEET` token price is 0.01u and `WBERA` token price is 7.5u. Assuming there are 150000 `YEET` tokens and 200 native `BERA` tokens in `StakeV2`, if admin tries to distribute the tokens as rewards, they must:

1. Call `executeRewardDistributionYeet()` to distribute `YEET` rewards, 75000 `YEET` tokens are swapped to `WBERA` tokens before depositing into Kodiak Vault. However, we cannot expect to receive exactly 100 `WBERA` tokens after the swap, because there is alway slippage loss, assuming the loss is 1 then the received `WBERA` token amount if 99;
2. Call `executeRewardDistribution` to distribute native `BERA` rewards, 100 `BERA` tokens are swapped to `YEET` tokens. Likewise, we may experience a slippage loss of 750 `YEET` tokens.

At last, the compounding vault shares will be minted but the transactions are actually unreasonable, why would we perform swaps when there is enough liquidity in contract? The slippage loss should've been avoided if we skip the swaps and deposit the tokens in `StakeV2` directly into Kodiak Vault.

**Please note the only purpose of swapping is to gain liquidity for depositing into Kodiak Vault, it's not a design choice or a trade off**. If there is sufficient liquidity in contract, we should use the tokens to avoid the slippage loss brought by the unnecessary swaps.

## Impact Details

Token swapping is performed even when there is sufficient liquidity in contract, this brings slippage loss and results in less rewards being distributed, causes damage to the protocol and users.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L145>

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L176>

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L193>

## Proof of Concept

## Proof of Concept

1. There are 150000 `YEET` tokens and 200 native `BERA` tokens in `StakeV2`;
2. To distributes the rewards, 75000 `YEET` tokens are swapped to 99 `WBERA` tokens, 100 native `BERA` tokens are swapped to 74250 tokens;
3. As a result, the actual tokens for reward distributes are 149250 (75000 + 74250) `YEET` tokens and 199 (99 + 100) `BERA` tokens, slippage loss is 15u;
4. By contrast, if we use the liquidity in the contract directly, we can avoid the slippage loss and get more rewards distributed.
