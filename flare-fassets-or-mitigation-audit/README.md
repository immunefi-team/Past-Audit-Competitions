# Flare FAssets | Mitigation Audit

## Reports by Severity

<details>

<summary>Low</summary>

* \#55242 \[SC-Low] `selfCloseExitTo` vulnerable to frontrunning griefing via `exit`
* \#55208 \[SC-Low] Executors receive a greater reward than the assigned value
* \#55002 \[SC-Low] Rewards claims increase pool collateral but do not notify `AssetManager` (stale CR/accounting after fix for #45893)
* \#54916 \[SC-Low] Minting cap can be surpassed via redemption fee

</details>

<details>

<summary>Insight</summary>

* \#54887 \[SC-Insight] Mitigation regression: pool token suffix length excludes valid 1- and 20-char values (the "fix" rejects valid edge-lengths and breaks agent creation)
* \#55230 \[SC-Insight] There is a Sub-gwei Executor-Fee can be Bypass and Freezes ETH in RedemptionRequests
* \#55241 \[SC-Insight] Insufficient validation of pool token suffix (allows consecutive hyphens) enables token symbol impersonation and user confusion
* \#55046 \[SC-Insight] Claimed rewards paid in legacy **wNat** after an upgrade are silently ignored by the balance-delta fix
* \#55174 \[SC-Insight] Over-assignment of payable in `claimAirdropDistribution` function could cause confusion regarding native token handling
* \#55025 \[SC-Insight] CoreVault refund failure can permanently freeze overpaid NAT on AssetManager
* \#54955 \[SC-Insight] Malicious Agents Can Trap Stakers by Raising Exit Collateral Ratio
* \#55049 \[SC-Insight] There is an issue related to msg.value Not Returned to Payer in Self-Close Exit

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#55242 \[SC-Low] `selfCloseExitTo` vulnerable to frontrunning griefing via `exit`
* \#54887 \[SC-Insight] Mitigation regression: pool token suffix length excludes valid 1- and 20-char values (the "fix" rejects valid edge-lengths and breaks agent creation)
* \#55208 \[SC-Low] Executors receive a greater reward than the assigned value
* \#55002 \[SC-Low] Rewards claims increase pool collateral but do not notify `AssetManager` (stale CR/accounting after fix for #45893)
* \#55230 \[SC-Insight] There is a Sub-gwei Executor-Fee can be Bypass and Freezes ETH in RedemptionRequests
* \#55241 \[SC-Insight] Insufficient validation of pool token suffix (allows consecutive hyphens) enables token symbol impersonation and user confusion
* \#55046 \[SC-Insight] Claimed rewards paid in legacy **wNat** after an upgrade are silently ignored by the balance-delta fix
* \#55174 \[SC-Insight] Over-assignment of payable in `claimAirdropDistribution` function could cause confusion regarding native token handling
* \#55025 \[SC-Insight] CoreVault refund failure can permanently freeze overpaid NAT on AssetManager
* \#54916 \[SC-Low] Minting cap can be surpassed via redemption fee
* \#54955 \[SC-Insight] Malicious Agents Can Trap Stakers by Raising Exit Collateral Ratio
* \#55049 \[SC-Insight] There is an issue related to msg.value Not Returned to Payer in Self-Close Exit

</details>
