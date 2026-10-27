# 51910 sc low inconsistent yield token transfer logic causes permanent loss of yield in distributeyield&#x20;

* Report ID: #51910
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief / Intro

When distributing yield by calling `ArcToken::distributeYield()` while `effectiveTotalSupply == 0`, the yield tokens are still sent to the contract even though there are no addresses to receive them. These tokens will be stuck in the contract.

### Vulnerability Details

Both yield distribution functions ensure `totalSupply()` is greater than zero (otherwise they revert). However, `effectiveTotalSupply` can still be zero when all addresses holding ArcTokens are yield-restricted.

* `ArcToken::distributeYieldWithLimit()` checks that `effectiveTotalSupply` is greater than zero before transferring the yield tokens to the contract, and returns early if it's zero.
* `ArcToken::distributeYield()` transfers the yield token amount to the contract first, then checks if `effectiveTotalSupply == 0` and may return early.

Because `distributeYield()` performs the token transfer before the `effectiveTotalSupply` check, tokens can be sent into the contract when there are no eligible recipients, leaving them permanently stuck. This behavior is inconsistent with `distributeYieldWithLimit()`.

## Impact Details

`ArcToken::distributeYield()` transfers the yield token amount to the contract even when `effectiveTotalSupply == 0`. This results in yield tokens being permanently stuck in the contract and is inconsistent with the guarded flow used in `ArcToken::distributeYieldWithLimit()`.

## References

Code references are provided throughout the original report and implementation:

* distributeYield source: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388-L460>
* totalSupply() check: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L397-L400>
* yield transfer in distributeYield: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L407-L408>
* effectiveTotalSupply early return in distributeYield: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L425-L428>
* distributeYieldWithLimit conditional transfer: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L518-L528>

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce distribution when no eligible holders exist

Call `ArcToken::distributeYield()` to distribute yield to token holders while skipping restricted accounts.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388-L460>

Assume there are currently no non-restricted holders.
{% endstep %}

{% step %}

### totalSupply() check passes and tokens are transferred

`totalSupply()` of ArcToken is greater than zero, so the call proceeds.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L397-L400>

The yield token amount is then sent to the contract:

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L407-L408>
{% endstep %}

{% step %}

### effectiveTotalSupply == 0 triggers early return but tokens are already stuck

After the transfer, the function checks `effectiveTotalSupply`. If it equals zero, the function returns early:

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L425-L428>

Because the tokens were transferred to the contract before this check, they remain stuck with no eligible recipients. This contrasts with `distributeYieldWithLimit()`, which performs the effective supply check before transferring tokens.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L518-L528>
{% endstep %}
{% endstepper %}

## Suggested Mitigation (non-exhaustive)

* Make the `effectiveTotalSupply == 0` check before performing any token transfers in `distributeYield()`, aligning its flow with `distributeYieldWithLimit()`.
* Alternatively, revert early (or skip transfer) if there are no eligible recipients to avoid tokens being stranded in the contract.
