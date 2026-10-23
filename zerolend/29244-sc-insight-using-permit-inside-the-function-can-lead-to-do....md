# 29244 - \[SC - Insight] Using permit inside the function can lead to Do...

Submitted on Mar 11th 2024 at 22:47:53 UTC by @stiglitz for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29244

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Temporary freezing of funds for at least 1 hour
* Permanent freezing of funds
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The abstract contract `BaseLPVault` implements the function `_takeTokens`.

```solidity
function _takeTokens(uint256 amount, PermitData memory permit) internal {
    if (permit.deadline > 0) {
        IERC2612(address(zero)).permit(
            msg.sender,
            address(this),
            permit.value,
            permit.deadline,
            permit.v,
            permit.r,
            permit.s
        );
    }
    zero.transferFrom(msg.sender, address(this), amount);
}
```

The function is not used anywhere because the codebase is not fully and completely implemented. However, if this function is called with `permit.deadline > 0`, the call can be front-runned by an attacker calling the permit function directly and causing DoS.

## Vulnerability Details

ERC2612 permit function is a permissionless function. Front-running direct permit calls is not a problem because the action of the function will be the same no matter who is the `msg.sender`

The problem is when the `permit` function is used as a part of another function. In this scenario, an attacker can monitor the mempool and if he spots the call that will execute the functions which contain `permit` call, he will extract signature data and call the `permit` function directly.

Now, because the signature is already used, it will revert the original transaction, and the code following `.permit` call won't be executed.

## Impact Details

DoS, which can be temporary or longer lasting if there is no walk-around for the specicif execution flow.

## References

It is exactly the scenario described by Trust: <https://www.trust-security.xyz/post/permission-denied>

## Proof of Concept

As it is a known issue, I believe the blog post from Trust Security (<https://www.trust-security.xyz/post/permission-denied>) is sufficient proof of the existence of the vulnerability. In case it is not I can provide PoC
