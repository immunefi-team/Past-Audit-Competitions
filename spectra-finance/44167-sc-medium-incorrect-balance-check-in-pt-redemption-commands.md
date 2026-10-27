# #44167 \[SC-Medium] Incorrect balance check in PT redemption commands

**Submitted on Apr 17th 2025 at 13:07:11 UTC by @Rhaydden for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44167
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Dispatcher contract incorrectly checks YT balances when users attempt to redeem PT, instead of checking the actual PT balances. This leads to a situation where users with legitimate redemption rights may be unable to redeem their PTs if the Dispatcher contract has fewer YTs than PTs, especially critical after maturity when YTs should no longer be required for redemption. This could result in permanent freezing of user funds in the protocol.

## Vulnerability Details

When handling `REDEEM_PT_FOR_ASSET` and `REDEEM_PT_FOR_IBT` commands, the code incorrectly limits the redemption amount based on the YT balance of the contract instead of the PT balance:

```solidity
// In REDEEM_PT_FOR_ASSET
shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this)));

// In REDEEM_PT_FOR_IBT
shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this)));
```

The Dispatcher should be checking the balance of the token being redeemed (PT) rather than a related token (YT). While PT and YT tokens are minted in equal amounts upon deposit and both are typically required for redemption before maturity, this balance check is fundamentally incorrect for several reasons:

1. The Dispatcher is acting as an intermediary that performs operations on behalf of users. When redeeming PTs, it should check that it has enough PTs to redeem, not YTs.
2. According to the protocol's design (as seen in PrincipalToken.sol's `_maxBurnable` function), after a PT reaches maturity, users can redeem PTs regardless of YT balances:

```solidity
 /**
     * @dev Computes the maximum amount of burnable shares for a user
     * @param _user The address of the user
     * @return maxBurnable The maximum amount of burnable shares
     */
    function _maxBurnable(address _user) internal view returns (uint256 maxBurnable) {
        if (block.timestamp >= expiry) {
            maxBurnable = balanceOf(_user);
        } else {
            uint256 ptBalance = balanceOf(_user);
            uint256 ytBalance = IYieldToken(yt).balanceOf(_user);
            maxBurnable = (ptBalance > ytBalance) ? ytBalance : ptBalance;
        }
    }
```

3. The router could end up with different amounts of PT and YT tokens, especially after maturity or through direct transfers. The current implementation would incorrectly limit redemptions in these scenarios.

## Impact Details

After a PT reaches maturity, users should be able to redeem their PTs regardless of YT balances. However, the Dispatcher would still limit redemptions based on YT balance, leaving PTs permanently frozen in the contract if YT balance is insufficient.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition//blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/tokens/PrincipalToken.sol#L908-L921>

<https://github.com/immunefi-team/Spectra-Audit-Competition//blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L247-L263>

## Proof of Concept

## Proof of concept

Consider a scenario:

1. A user deposits assets into Spectra and receives equal amounts of PT and YT tokens.
2. The user transfers some of their PTs to another wallet, but keeps their YTs in the original wallet.
3. The PT contract reaches maturity. According to the protocol design, PTs can now be redeemed without requiring YTs.
4. The user who received the PTs wants to redeem them through the Router using the REDEEM\_PT\_FOR\_ASSET command.
5. The user executes a transaction that calls the Router with the REDEEM\_PT\_FOR\_ASSET command.
6. Inside the Dispatcher's \_dispatch function, it checks:

```solidity
shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this)));
```

7. Since the Dispatcher has no YTs (or fewer YTs than PTs), this check evaluates to zero or a lower value than the PT amount being redeemed.
8. As a result, no redemption occurs (or less than the expected amount is redeemed).
9. The user's PTs are now effectively frozen in the Router contract, despite the PT contract allowing redemption after maturity without YTs.

## Fix

Consider replacing the YT balance check with a PT balance check in both REDEEM\_PT\_FOR\_ASSET and REDEEM\_PT\_FOR\_IBT commands:

```diff
// In REDEEM_PT_FOR_ASSET handler
- shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this)));
+ shares = Math.min(shares, IERC20(pt).balanceOf(address(this)));

// In REDEEM_PT_FOR_IBT handler
- shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this)));
+ shares = Math.min(shares, IERC20(pt).balanceOf(address(this)));
```
