# 90102 sc medium post slash deposit accounting in firelightvault leads to theft of recovered staker principal

**Submitted on Aug 23rd 2026 at 01:59:33 UTC by @calbebop for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90102
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

After a severe validated incident, `FirelightVault` can have zero underlying assets while slashed stakers still hold their original shares. Settlement closes the incident and immediately reopens public deposits. In that state, the vault’s share-conversion formula issues a new 1-wei depositor approximately the same number of shares as the entire pre-slash supply. When recovered principal is later returned to the vault, the new depositor can withdraw approximately half of it. In the PoC, a 1-wei post-settlement deposit diverts 5,000 FXRP out of a 10,000 FXRP recovery intended to restore the pre-slash staker.

## Vulnerability Details

`FirelightVault` prices deposits using:

```solidity
function _convertToSharesTotals(
    uint256 assets,
    uint256 totSupply,
    uint256 totAssets,
    Math.Rounding rounding
) private view returns (uint256) {
    return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
}
```

For this vault, `_decimalsOffset()` returns zero. A full slash can therefore leave:

```
totalAssets() = 0
totalSupply() = S > 0
```

A public 1-wei deposit is then minted as:

```
1 * (S + 1) / (0 + 1) = S + 1 shares
```

The new post-slash depositor consequently owns approximately half of all outstanding shares before recovered value is credited to the vault.

This state is reachable through ordinary incident settlement. `IncidentManager.approveCurrentAssessment()` executes payout, marks the incident closed, and decrements the active incident count:

```solidity
incident.status = IncidentStatus.CLOSED;
_decreaseIncident(incident.period);
```

When the count reaches zero, `_decreaseIncident()` clears the vault’s incident flag, immediately allowing deposits again.

Firelight’s documentation is consistent with this sequence:

* slashed stakers retain their original share balances;
* the vault’s redemption value falls after the slash;
* deposits reopen when the slash settles;
* recovered assets flow back through the loss waterfall in reverse order, with slashed stakers made whole first;
* liquidation overperformance is returned to the vault rather than increasing payouts.

The existing zero-share deposit guard does not prevent this case. A 1-wei deposit in the zero-asset/nonzero-supply state does not round to zero shares; it receives `S + 1` shares.

The issue is not limited to an exact zero-asset state. After a partial slash leaving residual assets `A`, an attacker who deposits `A` receives approximately `S` additional shares and captures approximately half of a later recovery. This was reproduced for slash severities from 25% through 100%.

A direct mitigation is to reject deposits while `totalAssets() == 0 && totalSupply() > 0`. A complete fix must also prevent post-slash deposits from sharing value earmarked to restore pre-slash shareholders—for example, by retiring or loss-adjusting slashed shares, migrating to a new share epoch, or distributing recovery against a pre-incident shareholder snapshot before deposits reopen.

## Impact Details

The selected impact is direct theft of user funds other than unclaimed yield.

In the full-slash PoC:

| State                                      |            Result |
| ------------------------------------------ | ----------------: |
| Original staker principal                  |       10,000 FXRP |
| Vault assets after legitimate payout       |                 0 |
| Outstanding shares after payout            |          10,000e6 |
| Attacker deposit                           |             1 wei |
| Shares received by attacker                |      10,000e6 + 1 |
| Recovered principal returned to vault      |       10,000 FXRP |
| Attacker withdrawable value                | 5,000.000001 FXRP |
| Attacker net profit                        |        5,000 FXRP |
| Pre-slash staker recovery without attacker |       10,000 FXRP |
| Pre-slash staker recovery with attacker    |        5,000 FXRP |

Without the attacker, the returned 10,000 FXRP restores the original staker’s full redeemable value. With the attacker, half of that recovery is transferred to an account that contributed only 1 wei after the loss.

The diverted value is recovered principal, not unclaimed yield: it is the FXRP previously slashed from the staker and returned through the documented recovery path.

The attacker is not the cover buyer and holds no privileged role. After settlement, the attack uses only the public vault functions `deposit()`, `withdraw()`, and `claimWithdraw()`. The incident and payout are handled through normal, non-malicious protocol operation.

## References

* [FirelightVault.deposit()](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L673)
* [FirelightVault.\_convertToSharesTotals()](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1074)
* [FirelightVault.totalAssets()](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L423-L429)
* [IncidentManager.approveCurrentAssessment()](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438)
* [IncidentManager active-incident/deposit-gate logic](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L805-L830)
* [Risk and Exposure documentation](https://docs.firelight.finance/for-stakers/risk-and-exposure.md)
* [Claims Liquidation documentation](https://docs.firelight.finance/protocol-architecture/claims-liquidation.md)
* [Periods and Coverage Cycles documentation](https://docs.firelight.finance/core-concepts/periods-and-coverage-cycles.md)

## Proof of Concept

## Proof of Concept

The following was reproduced on a local Hardhat network using the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, `IncidentManager`, price-feed adapter, premium token, first-loss token, and vault asset. No public network or production service was tested.

### 1. Deploy the real Phase 2 integration

Deploy and connect the real contracts with their intended roles:

* `FirelightVault`
* `CoverOrderAllocator`
* `CoverNFT`
* `IncidentManager`
* vault asset
* premium token
* first-loss token
* price-feed adapter

Grant the incident manager its required vault roles and authorize the configured payout receiver.

### 2. Fund the documented first-loss layer

Fund the first-loss wallet with `1,000` first-loss tokens and authorize `IncidentManager` to spend them.

### 3. Deposit the original staker principal

An original staker deposits `10,000 FXRP`:

```
vault.totalAssets()       == 10,000e6
vault.totalSupply()       == 10,000e6
oldStaker share balance   == 10,000e6
oldStaker maxWithdraw()   == 10,000e6
```

### 4. Settle genuine cover

A cover buyer separate from the attacker purchases `11,000 USD` of cover.

The curator creates the order, the allocator commits and settles it, and the cover buyer receives CoverNFT token 0.

### 5. Approve a legitimate incident

Create, confirm, assess, submit, and approve a genuine incident for the full `11,000 USD` covered loss.

The payout consumes the funded `1,000` first-loss tokens first, then pays the remaining `10,000 FXRP` from the vault.

### 6. Verify the post-slash state

Immediately after approval:

```
firstLossToken.balanceOf(payoutReceiver) == 1,000e6
asset.balanceOf(payoutReceiver)          == 10,000e6

vault.totalAssets()                      == 0
vault.totalSupply()                      == 10,000e6
asset.balanceOf(vault)                   == 0
vault.maxWithdraw(oldStaker)             == 0
vault.hasActiveIncident(currentPeriod)   == false
```

The incident has closed normally and public deposits are open again.

### 7. Verify that the attacker is unprivileged

The PoC checks that the attacker holds none of the relevant roles on the vault, allocator, or incident manager, including:

* vault `DEFAULT_ADMIN_ROLE`
* vault `PAYOUT_ROLE`
* vault `INCIDENT_ROLE`
* vault `CHECKPOINT_ROLE`
* allocator `DEFAULT_ADMIN_ROLE`
* allocator `CURATOR_ROLE`
* allocator `ALLOCATOR_ROLE`
* incident manager `DEFAULT_ADMIN_ROLE`
* incident manager `CURATOR_ROLE`
* incident manager `ASSESSMENT_APPROVER_ROLE`

The attacker also has no ownership or beneficiary role in the settled cover.

### 8. Make the 1-wei post-slash deposit

The attacker approves and deposits `1 wei`:

```
attacker deposit       == 1
attacker share balance == 10,000e6 + 1
vault.totalSupply()    == 2 * 10,000e6 + 1
vault.totalAssets()    == 1
```

### 9. Return the exact recovered principal

The payout receiver transfers the exact `10,000 FXRP` previously slashed from the vault back to the vault.

This represents the documented return of recovered vault-asset value to the vault layer. No new value is minted, and the recovered amount is the same principal previously removed from the pre-slash staker.

### 10. Verify the resulting ownership split

After the recovery transfer:

```
vault.totalAssets()              == 10,000e6 + 1
attacker.maxWithdraw(attacker)  == 5,000e6 + 1
oldStaker.maxWithdraw()         == 5,000e6
```

### 11. Complete the normal delayed withdrawal

The attacker requests:

```solidity
vault.withdraw(5_000e6 + 1, attacker, attacker);
```

The vault records a withdrawal request for the next period. After that withdrawal bucket becomes claimable, the attacker calls:

```solidity
vault.claimWithdraw(withdrawPeriod);
```

The vault transfers `5,000e6 + 1` FXRP to the attacker.

### 12. Verify the final state

After the claim:

```
attacker token balance increase == 5,000e6 + 1
attacker net profit             == 5,000e6

vault.totalAssets()             == 5,000e6
vault.totalSupply()             == 10,000e6
vault.pendingWithdrawAssets()   == 0
oldStaker.maxWithdraw()         == 5,000e6
```

FXRP conservation is also checked:

```
payoutReceiver FXRP + vault FXRP + attacker FXRP
    == original staker principal + attacker’s 1-wei deposit
```

### Negative control

Repeating the same recovery without the attacker’s 1-wei deposit restores the original staker’s full claim:

```
vault.totalAssets()       == 10,000e6
oldStaker.maxWithdraw()   == 10,000e6
```

Thus, the attacker’s post-settlement 1-wei deposit is the sole change that converts a full 10,000 FXRP recovery into a 5,000 FXRP recovery for the slashed staker.

### Partial-slash control

The same behavior was tested after ordinary partial slashes:

* 25%
* 50%
* 75%
* 90%
* 99%
* 100%

When the vault has residual assets `A`, an attacker depositing `A` after settlement receives approximately the old share supply and captures approximately half of the later recovery.

For example, after a 90% slash:

```
residual vault assets       == 1,000 FXRP
attacker deposit            == 1,000 FXRP
recovered principal         == 9,000 FXRP
attacker withdrawal         == 5,499.999997 FXRP
attacker net profit         == 4,499.999997 FXRP
captured recovery           ~= 50%
```

### Routing control

The same result was also reproduced when the recovered FXRP was routed through the real `VaultRewardDistributor` instead of being transferred directly to the vault. The distributor was used only as an alternate asset-forwarding path; the source of the funds remained the previously slashed principal.
