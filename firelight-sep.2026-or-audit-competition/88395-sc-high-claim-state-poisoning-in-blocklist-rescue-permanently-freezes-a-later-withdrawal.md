# 88395 sc high claim state poisoning in blocklist rescue permanently freezes a later withdrawal

**Submitted on Aug 13th 2026 at 00:59:52 UTC by @jerrysd for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88395
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted()` permanently marks the source `(period, account)` as claimed. If the account is unblocked before that period rolls over, `redeem()` and `withdraw()` still accept a new request for the same future period and burn the user's shares. At maturity, both the normal claim and the only rescue path reject the newly recorded entitlement with `AlreadyClaimedPeriod`. The accepted principal is permanently frozen in the vault.

## Vulnerability Details

The rescue path moves all current pool shares away from `from` and sets a terminal flag:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

However, `_requestWithdraw()` never checks that flag before adding new shares to the same key:

```solidity
uint256 period = currentPeriod() + 1;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

The claim and rescue exits both reject that new balance:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
// ...
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

## Impact Details

An unblocked user can lose any new principal withdrawn to the poisoned account during the same vault period. The vault burns the shares and retains the corresponding FXRP, but exposes no call that can claim, move, cancel, or reset the entitlement. The demonstrated 60 FXRP is arbitrary and scales to the user's remaining position.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>

## Link to Proof of Concept

<https://gist.github.com/JRBusiness/dfeb73aeb4042fe4a9beeedd57ecd3da>

## Proof of Concept

## Proof of Concept

Run from the frozen `v1_audit_ready` branch:

```bash
npx hardhat test --no-compile rescuedWithdrawalClaimPoison.js
```

The test deposits 100 FXRP, rescues a 40 FXRP future withdrawal through the documented blocklist flow, unblocks the user, and has the user redeem its remaining 60 FXRP into the same future period. The 40 FXRP rescue recipient claims successfully. The 60 FXRP later request remains recorded and funded, but both recovery paths revert:

```
[claim-poison setup] period=1 firstRescued=40000000 laterAccepted=60000000 sourceClaimed=true
[claim-poison maturity] rescuedClaim=40000000 frozen=60000000 pending=60000000 vaultBalance=60000000
[claim-poison claim] reverted=AlreadyClaimedPeriod
[claim-poison recovery] rescue-retry=reverted AlreadyClaimedPeriod
1 passing
```
