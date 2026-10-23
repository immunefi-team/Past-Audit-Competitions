# 50937 sc medium non zero approve pattern causes permanent freeze of token deposits e g usdt due to erc20 incompatibility

**Submitted on Jul 29th 2025 at 19:33:23 UTC by @Bug82427 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50937
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The DexAggregatorWrapperWithPredicateProxy contract uses the ERC20 `approve()` function to grant allowance to external swap/bridge routers, but it does so without first setting the allowance to zero. Tokens like USDT enforce a non-standard rule where `approve(from, spender, newAmount)` will revert if `spender` already has a non-zero allowance. This creates a situation where one user's successful deposit causes the token allowance to become locked, and all future attempts to deposit that token will revert, freezing the token permanently for every user.

### Vulnerability Details

In both `_oneInchHelper` and `_okxHelper`, the contract performs approvals like:

```solidity
depositAsset.safeApprove(address(aggregator), depositAmount);
...
supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);
```

`safeApprove` here is from Solmate’s SafeTransferLib and directly calls:

```solidity
require(token.approve(spender, amount), "APPROVE_FAILED");
```

This will succeed only if the current allowance is 0 or the token allows non-zero → non-zero changes (most standard tokens do; USDT does not).

The critical flaw is:

* On the first deposit, `approve(spender, amount)` works because allowance is 0.
* On the second deposit, even with a new amount, `approve(spender, amount)` will fail unless allowance is first set to 0.

This breaks compatibility with major real-world tokens like:

* USDT: enforces `require(oldAllowance == 0 || newAllowance == 0)`
* KNC (old) and some TrueUSD variants
* Custom bridged tokens on L2s with wrapper logic

Once a single deposit with such a token succeeds, future deposits will always revert. There’s no recovery mechanism or token-specific handling logic, so this becomes a permanent freeze for that token on the wrapper.

### Impact Details

This causes a Permanent Freezing of Funds scenario:

* A single user’s successful deposit “poisons” the contract state by leaving a non-zero allowance.
* Every future deposit involving that token will revert.
* Affected tokens (like USDT) are widely used and often part of default configurations in aggregators.
* It doesn’t just block an attacker—it blocks all future users from using that token in any function path (`depositOneInch`, `depositOkxUniversal`, etc.)
* Users may be forced to use alternate tokens or redeploy the wrapper, which is impractical.
* This issue is particularly dangerous in production where:
  * Protocols support arbitrary user-chosen tokens
  * Vaults or strategies are configured to accept USDT or other non-compliant tokens
  * Routing logic (e.g. in 1inch or OKX) defaults to USDT in many paths
* There is no user-visible indication of why the call fails. The result is a complete and silent freeze of deposits for that token.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Alice makes a valid USDT deposit

```solidity
USDT.approve(address(wrapper), type(uint256).max);

wrapper.depositOneInch(
    USDT,
    teller,
    100e6, // 100 USDT
    executor,
    desc, // USDT -> supportedAsset swap
    data,
    0,
    predicateMessage
);
```

This call succeeds.

Inside `_oneInchHelper`, the contract transfers 100 USDT from Alice, then calls:

```solidity
USDT.approve(address(aggregator), 100e6); // Allow aggregator to pull funds
```

Since allowance was 0 before, this succeeds.
{% endstep %}

{% step %}

### Step

Bob attempts to deposit USDT

Bob tries the same flow:

```solidity
USDT.approve(address(wrapper), type(uint256).max);

wrapper.depositOneInch(
    USDT,
    teller,
    50e6, // 50 USDT
    executor,
    desc,
    data,
    0,
    predicateMessage
);
```

This time, `_oneInchHelper` does:

```solidity
USDT.approve(address(aggregator), 50e6); // Allowance already > 0
```

This reverts with:

```
"APPROVE_FAILED"
```

Result: Bob's deposit fails. So will Charlie’s, Dave’s, etc. Wrapper is now permanently broken for USDT.
{% endstep %}

{% step %}

### Step

No workaround exists

There is no logic to reset the allowance to 0 before setting it again.

Only ways to unstick it:

* Deploy a new wrapper (inconvenient)
* Manually zero allowance (not possible without internal logic)

Conclusion: This is a widespread, real-world problem that breaks compatibility with one of the most-used stablecoins (USDT). It requires no privileged role, no special conditions—just two normal users trying to deposit the same token.
{% endstep %}
{% endstepper %}
