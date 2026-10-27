# 90489 sc low firelightvault maxmint returns a share count larger than mint will accept violating eip 4626 s must not be higher than the actual maximum the standard mint maxmint u u in&#x20;

**Submitted on Aug 25th 2026 at 09:20:33 UTC by @magento for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90489
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`FirelightVault.maxMint()` computes its answer in **share space** while `mint()` enforces the deposit limit in **asset space**. Because the share/asset conversion is affine rather than linear, the two disagree, and `maxMint()` returns a value strictly larger than `mint()` will accept. EIP-4626 states that `maxMint` "MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary)". It overestimates. The canonical integration `mint(vault.maxMint(u), u)` therefore reverts `DepositLimitExceeded` — measured at **49% of consecutive deposit-limit values** at a 2.0 share price. No role is involved at any point: the deposit limit is set at `initialize()`, the share price arises from ordinary yield or an unprivileged token transfer, and the failing call is made by an ordinary user. The sibling function `maxDeposit()` performs the same job in asset space and is exact, which is what makes this an arithmetic error rather than a design choice.

## Vulnerability Details

**No privileged actor participates in this report.** I state this first because it is the axis on which reports are commonly closed. There is no role in the reproduction after deployment, no actor behaving incorrectly, and no trusted party whose discretion is being second-guessed. The defect is a wrong number returned by a `view` function, provable by arithmetic and reproducible in a single transaction by any address.

**Nor is this a missing check.** The check exists, is implemented, and is enforced. `maxMint` is an `override` that Firelight wrote deliberately; the claim is that the value it computes is incorrect.

**The two functions, side by side.** `contracts/core/FirelightVault.sol:371-378`:

```solidity
    function maxDeposit(address receiver) public view override returns (uint256 amount) {
        uint256 assets = totalAssets();
        if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) {
            return 0;
        } else {
            return depositLimit - assets;          // ASSET space
        }
    }
```

`contracts/core/FirelightVault.sol:385-393`:

```solidity
    function maxMint(address receiver) public view override returns (uint256 amount) {
        uint256 shares = totalSupply();
        uint256 sharesLimit = convertToShares(depositLimit);
        if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || shares > sharesLimit) {
            return 0;
        } else {
            return sharesLimit - shares;           // SHARE space
        }
    }
```

The limit they must both respect is enforced in **asset** space, `contracts/core/FirelightVault.sol:984-987`:

```solidity
        _totalSupply += shares;
        _totalAssets += assets;

        if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

`maxDeposit` subtracts in the same space the check uses, so it is exact. `maxMint` **converts first and subtracts second**, and the conversion is affine — `contracts/core/FirelightVault.sol:1064`:

```solidity
        return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
```

Because of the `+1` virtual offsets, `convertToShares(L) - S != convertToShares(L - A)`. The residual is not a rounding artifact; it is a systematic term that grows with the share price, which is why the failure rate tracks `1 - 1/price` exactly (measured below).

**The specification this violates.** EIP-4626, `maxMint`:

> "MUST return the maximum amount of shares `mint` would allow to be deposited to `receiver` and not cause a revert, **which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary)**."

The contract declares conformance by overriding the interface function, and the OpenZeppelin `IERC4626` NatSpec vendored in this repository restates the same obligation at `node_modules/@openzeppelin/contracts/interfaces/IERC4626.sol:112-115`. The returned value is higher than the accepted maximum — the exact direction the standard forbids, in the one place it explicitly tells implementers to err the other way.

**Relationship to OpenZeppelin CR-01 — distinct, and the reason this exists.** The OpenZeppelin review (2025-07-21) raised **CR-01, "Limit Functions Not Overridden to Enforce Deposit/Withdraw Restrictions"**:

> "The FirelightVault inherits from ERC4626 ... However, none of these functions are currently overridden to account for **deposit limits, user block status, or whether the contract is paused**. Consider overriding them ... **Update: Resolved in pull request #9 at commit ddb821b.**"

CR-01 is about the **boolean predicates** — that no override existed. It is **Resolved**, so it is not an unpatched disclosed issue. The remediation added both overrides and got `maxDeposit` arithmetically right and `maxMint` arithmetically wrong. This report concerns the arithmetic of that remediation, not the predicates CR-01 named — the predicates in `maxMint` are correct.

**`maxWithdraw` and `maxRedeem` do not share the defect** and I verified it rather than assuming: `maxWithdraw` returns `_convertToAssets(balanceOf(owner), Floor)` (`:405`) and `withdraw` re-derives `ceil(assets/r)`, with `ceil(floor(b*r)/r) <= b` in both price regimes.

## Impact Details

**In-scope impact: "Contract fails to deliver promised returns, but doesn't lose value" (Low).**

No funds are lost or stolen. What fails is a property the contract states by implementing a standard: `maxMint(u)` is the value an integrator is told it may mint, and minting it reverts.

Measured, at a 2.0 share price, sweeping consecutive `depositLimit` values: **20 of 41 revert (49%)**, against `1 - 1/price = 50%` predicted from the arithmetic. The rate rises with the share price — approximately 9% at 1.1x, 50% at 2.0x, 90% at 10x — and the share price exceeds 1 as soon as any yield accrues, which is the vault's normal operating state. This is not a boundary artifact and not a single unlucky configuration.

The affected pattern, `mint(vault.maxMint(user), user)`, is the canonical way a router, zapper, aggregator or vault-of-vaults deposits a maximal position, precisely because the standard promises the returned value will not revert. Every such caller fails, permanently, with no way to discover the correct amount other than to guess downward. The workaround — `maxMint() - 1` — is not derivable from the interface and is itself only correct because the overstatement happens to be one unit at this price; the residual `(A - S)/(S + 1)` grows with the share price.

`maxMint` and `maxDeposit` have no internal consumers in the protocol (verified by grep across `contracts/core` and `contracts/oracle`), so the blast radius is entirely external integrators — but that is exactly the audience an ERC4626 view function exists to serve.

## References

* `contracts/core/FirelightVault.sol:385-393` — `maxMint`, computing in share space
* `contracts/core/FirelightVault.sol:371-378` — `maxDeposit`, computing in asset space and exact
* `contracts/core/FirelightVault.sol:984-987` — the asset-space enforcement both must respect
* `contracts/core/FirelightVault.sol:1064` — `_convertToSharesTotals`, the affine conversion
* `contracts/core/FirelightVault.sol:225` — `depositLimit` set at `initialize`, so no role is needed
* `contracts/core/FirelightVault.sol:405` — `maxWithdraw`, checked and NOT affected
* `node_modules/@openzeppelin/contracts/interfaces/IERC4626.sol:112-115` — the vendored obligation
* EIP-4626 — `maxMint` "MUST NOT be higher than the actual maximum that would be accepted"
* OpenZeppelin Firelight Vault audit 2025-07-21, **CR-01** — Resolved in PR #9 / `ddb821b`

## Link to Proof of Concept

<https://gist.github.com/MihRazvan/ad7d71a904217881ef16c828ef9a610f>

## Proof of Concept

## Proof of Concept

Environment: frozen contest commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, Hardhat, local only. Real `FirelightVault` behind the repository's own `deployVault` fixture — no mock is substituted for the contract under test.

**Zero privileged actions.** `depositLimit` is supplied to `initialize()` (`FirelightVault.sol:225`, `_updateDepositLimit(initParams.depositLimit)`), so no `DEPOSIT_LIMIT_UPDATE_ROLE` call is required. Every other step is an ordinary user action.

Setup — `depositLimit = 299`:

1. User0 deposits 100 assets. (ordinary `deposit`, no role)
2. User2 sends 100 assets to the vault with a plain ERC20 `transfer`. (no vault interaction, no role) The vault now holds `totalAssets = 200` against `totalSupply = 100` — a 2.0 share price. Ordinary yield accrual produces the same state; the transfer is simply the shortest way to reach it.

TARGET:

```
  depositLimit             : 299
  totalAssets / totalSupply: 200 / 100      -> share price 2.0
  maxMint(r)               : 50             <- what the vault SAYS is mintable
  previewMint(maxMint(r))  : 100            <- assets mint() will actually pull
  headroom to depositLimit : 99
  overstatement            : 1 asset unit
  mint(maxMint(r), r)      -> REVERTS DepositLimitExceeded
```

CONTROL A — the identical call with one fewer share succeeds: `mint(maxMint(r) - 1, r)` → success. The function is wrong by exactly the overstatement.

CONTROL B — the sibling is exact. `deposit(maxDeposit(r), r)` → success, and `totalAssets()` lands on `depositLimit` precisely. The protocol computes this correctly one function earlier; only `maxMint` converts before subtracting.

CONTROL C — isolating the share price. Identical script with the donation removed, so the price is exactly 1.0: `mint(maxMint(r), r)` → success. The single differing input is the share price.

RATE — proving it is not a chosen boundary. Sweeping 41 consecutive `depositLimit` values at a fixed 2.0 share price:

```
  mint(maxMint()) REVERTED for 20 of 41  = 49%
  predicted by (1 - 1/price)             = 50%
```

RECOMMENDED FIX — compute in the same space the check uses, mirroring `maxDeposit`:

```solidity
        return convertToShares(depositLimit - totalAssets());
```

or equivalently return `previewWithdraw`-style the largest `s` with `previewMint(s) <= depositLimit - totalAssets()`. Either keeps the standard's "underestimate if necessary" direction.

A runnable Hardhat test containing the TARGET, all three controls and the rate sweep is provided in the linked secret Gist. EOF wc -l /Users/razvan/Security\_Repos/2026-08-firelight/submissions/F-005-SUBMISSION.md
