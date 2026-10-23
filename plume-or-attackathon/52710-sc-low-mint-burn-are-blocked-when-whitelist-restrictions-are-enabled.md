# 52710 sc low mint burn are blocked when whitelist restrictions are enabled

**Submitted on Aug 12th 2025 at 15:03:43 UTC by @IronsideSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52710
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

When transfer restrictions are enabled for an `ArcToken` using the whitelist module, minting and burning always revert. The restriction module requires both `from` and `to` addresses to be whitelisted when `transfersAllowed == false`. Because `address(0)` cannot be whitelisted, mints (`from == address(0)`) and burns (`to == address(0)`) are blocked. To mint or burn, the admin would have to temporarily set `transfersAllowed = true`, which enables transfers for everyone and defeats the purpose of the restriction regime.

### Vulnerability Details

* Resulting behavior under restriction:
  * Mint reverts: `isTransferAllowed(address(0), recipient)` is false because `address(0)` is not whitelisted.
  * Burn reverts: `isTransferAllowed(holder, address(0))` is false because `address(0)` is not whitelisted.
* PoC alignment:
  * `test_MintReverts_WhenTransfersRestricted` and `test_BurnReverts_WhenTransfersRestricted` in `test/WhitelistMintBurnPoC.t.sol` both pass:
    * After `setTransfersAllowed(false)`, `ArcToken.mint(...)` and `ArcToken.burn(...)` revert with `TransferRestricted`.
  * Your trace logs confirm the reverts with the module returning `false` for zero-address endpoints.
* Why toggling is not an option:
  * Admin could disable restrictions to perform mint/burn; however, setting `transfersAllowed = true` permits all transfers during the window, allowing unrestricted movement that violates the token’s intended compliance or freeze policy.
* The whitelist module enforces that both endpoints must be whitelisted when restrictions are active:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L101-L111>

```solidity
File: arc/src/restrictions/WhitelistRestrictions.sol

104:     function isTransferAllowed(address from, address to, uint256 /*amount*/ ) external view override returns (bool) {
105:         WhitelistStorage storage ws = _getWhitelistStorage();
108:         if (ws.transfersAllowed) {
109:             return true;
110:         }
111: 
113:   >>>   return ws.isWhitelisted[from] && ws.isWhitelisted[to];
114:     }

```

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/WhitelistRestrictions.sol#L135-L137>

```solidity
File: arc/src/restrictions/WhitelistRestrictions.sol

135:     function addToWhitelist(
136:         address account
137:     ) external onlyRole(MANAGER_ROLE) {
138:   >>>   if (account == address(0)) {
139:             revert InvalidAddress();
140:         }
141: 
142:         WhitelistStorage storage ws = _getWhitelistStorage();
143:         if (ws.isWhitelisted[account]) {
144:             revert AlreadyWhitelisted(account);
145:         }
146: 
147:         ws.isWhitelisted[account] = true;
148:         ws.whitelistedAddresses.add(account);
    ---- SNIP ----
151:     }

```

* The token consults restriction modules on every state change, including mint (`from == address(0)`) and burn (`to == address(0)`):

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L663-L682>

```
File: attackathon-plume-network/arc/src/ArcToken.sol

140:     function _update(address from, address to, uint256 amount) internal virtual override {
    ---- SNIP ----
150:         address specificTransferModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
151:         if (specificTransferModule != address(0)) {
152:             transferAllowed =
153:   >>>           transferAllowed && ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
154:         }
155: 
156:         address globalTransferModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
157:         if (globalTransferModule != address(0)) {
158:   >>>       try ITransferRestrictions(globalTransferModule).isTransferAllowed(from, to, amount) returns (
159:                 bool globalAllowed
160:             ) {
161:                 transferAllowed = transferAllowed && globalAllowed;
162:             } catch {
163:                 transferAllowed = false;
164:             }
165:         }
    ---- SNIP ----
199:     }
```

### Impact Details

* Denial of administrative functions:
  * Admin cannot mint or burn while restrictions are enforced, effectively locking operational control over supply.
* Policy violation pressure:
  * To proceed with mint/burn, admin must open transfers, enabling everyone to move tokens during that period—contrary to the design intent of restricting transfers.
* Production risk:
  * In regulated/whitelisted deployments, this either forces a permanent freeze of supply ops or creates unsafe windows of unrestricted movement.

### References

See code snippets with github links in `Vulnerability Details` section above

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/d2d601e05c42c9815c9daa5a7eaacca4>

## Proof of Concept

## Proof of Concept

Follow steps in <https://gist.github.com/IronsideSec/d2d601e05c42c9815c9daa5a7eaacca4>
