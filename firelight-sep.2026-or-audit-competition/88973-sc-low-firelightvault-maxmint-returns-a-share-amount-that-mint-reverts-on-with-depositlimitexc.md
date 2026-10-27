# 88973 sc low firelightvault maxmint returns a share amount that mint reverts on with depositlimitexceeded because the two functions round in opposite directions across depositlimit

**Submitted on Aug 16th 2026 at 05:03:12 UTC by @Darkseid for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88973
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

EIP-4626 requires maxMint to return the maximum share amount mint will accept, and states that mint MUST NOT revert for any value at or below it. In FirelightVault the two functions disagree: maxMint floors its conversion from assets to shares, while mint converts back with ceiling rounding and only then compares the result against depositLimit. Floor-then-ceil across a division is not the identity, so the two differ by up to one unit and mint(maxMint(receiver), receiver) reverts. The same asymmetry breaks deposit(maxDeposit(receiver), receiver) at the opposite end of the range.

## Vulnerability Details

maxMint converts the deposit limit to shares with floor rounding. OpenZeppelin's convertToShares uses Math.Rounding.Floor:

```solidity
// contracts/core/FirelightVault.sol:385-393
function maxMint(address receiver) public view override returns (uint256 amount) {
    uint256 shares = totalSupply();
    uint256 sharesLimit = convertToShares(depositLimit);
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || shares > sharesLimit) {
        return 0;
    } else {
        return sharesLimit - shares;
    }
}
```

mint converts the caller's share amount back to assets with ceiling rounding, then enforces the limit on that result:

```solidity
// contracts/core/FirelightVault.sol:698-702
(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares,
    false,
    Math.Rounding.Ceil
);
```

```solidity
// contracts/core/FirelightVault.sol:987
if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

Worked example with depositLimit = 20, totalAssets A = 7, totalSupply S = 3:

```
maxMint  = floor(20 * (3+1) / (7+1)) - 3 = floor(80/8) - 3 = 10 - 3 = 7
mint(7)  -> assets = ceil(7 * (7+1) / (3+1)) = ceil(56/4) = 14
            limit check: 7 + 14 = 21 > 20  ->  DepositLimitExceeded
```

The A > S state, meaning share price above 1, is reached by any asset arriving outside the deposit/withdraw flow. VaultRewardDistributor.distributeRewards transfers directly to the vault (VaultRewardDistributor.sol:146), and a plain donation does the same.

Symmetric case on the deposit side: maxDeposit returns depositLimit - totalAssets(), but deposit() rejects the call when the resulting shares floor to zero (L667-669). With depositLimit = 2, A = 1, S = 0, maxDeposit returns 1 and deposit(1) reverts InvalidAmount.

Root cause: the two directions of the conversion use opposite rounding modes, and the limit is enforced on the ceiling side while advertised from the floor side.

## Impact Details

Preconditions: share price above 1 (any forwarded reward or donation) and totalAssets near depositLimit. No privileged role, no attacker, no unusual sequencing.

Impact: any integrator following the standard "read the max, then act on the max" pattern reverts. That is the canonical vault-integration idiom. Aggregators, zappers, routers and automated LP strategies all use it, and it is the pattern EIP-4626's maxMint contract exists to support. The failure is a hard revert, so the consequence is denial of the deposit path for conforming integrators rather than a loss of funds.

Who bears it: integrating protocols and the LPs routing through them, who cannot deposit at the top of the limit range despite the vault advertising capacity.

Why Low and not higher: no funds are at risk and no accounting is corrupted. This is a standards-conformance defect whose consequence is an integration failure, which is why it maps to "fails to deliver promised returns, but doesn't lose value" rather than any loss-of-funds impact.

## Not a Duplicate / Not a Known Issue

None of the three declared known issues concerns EIP-4626 conformance or the deposit limit. K1 and K2 are first-loss-buffer items and K3 is the aggregate payout cap. The published Phase 1 finding set contains no maxMint or maxDeposit entry.

## Link to Proof of Concept

<https://gist.github.com/morpheus-csmith/624dbd2c9fa126ea92d170b395f48287>

## Proof of Concept

### Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

```
# Save the attached PoC file as test/audit_pocs.js
#   sha256: 64529474656a335ff4d4cbb07d3a85300f0a41ffad252cfed656094e24952716
# The file contains all PoCs from this submission set; this report's is PoC2.
npx hardhat test test/audit_pocs.js --grep "PoC2"
```

### Test

This finding's test, as it appears in the attached audit\_pocs.js:

```javascript
// Excerpt of test/audit_pocs.js (sha256 64529474...2716).
// The full attached file contains all four PoCs; this is PoC2.

const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

const tinyVault = () =>
  loadFixture(deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 20n }));

describe("AUDIT PoCs", function () {
  it("PoC2: mint(maxMint(user)) reverts with DepositLimitExceeded", async () => {
    const { firelight_vault, token_contract, users, utils } = await tinyVault();
    const user = users[0];

    await utils.mintAndApprove(40n, user);
    await firelight_vault.connect(user).deposit(3n, user.address);

    await token_contract.mintTo(firelight_vault.target, 4n);

    expect(await firelight_vault.totalAssets()).to.equal(7n);
    expect(await firelight_vault.totalSupply()).to.equal(3n);

    const maxShares = await firelight_vault.maxMint(user.address);
    expect(maxShares).to.equal(7n);

    await expect(
      firelight_vault.connect(user).mint(maxShares, user.address)
    ).to.be.revertedWithCustomError(firelight_vault, "DepositLimitExceeded");
  });
});
```

### Output

```
AUDIT PoCs
  ✔ PoC2: mint(maxMint(user)) reverts with DepositLimitExceeded (821ms)

1 passing (823ms)
```

Environment: Ubuntu (WSL2), Node v22.20.0, npm 10.9.3, Hardhat 2.22.19, solc 0.8.28 (optimizer on, runs: 800, viaIR, the repository's own production setting for this contract). The repository's existing suite passes on the same tree (npx hardhat test test/vault\_payout.js gives 4 passing), so this result is measured against a known-good baseline.
