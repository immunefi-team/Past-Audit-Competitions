# 90153 sc low cr 01 fix bypass maxmint converts the absolute cap instead of remaining capacity causing mint maxmint to revert after rewards

## #90153 \[SC-Low] CR-01 fix bypass: maxMint() converts the absolute cap instead of remaining capacity, causing mint(maxMint) to revert after rewards

**Submitted on Aug 23rd 2026 at 11:20:19 UTC by @crypt0n1337 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90153
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Bug Description

### Brief/Intro

`FirelightVault.maxMint()` calculates the remaining share capacity as `convertToShares(depositLimit) - totalSupply()`. This subtracts shares only after converting the **absolute** asset cap at the current exchange rate. ERC-4626 conversion includes virtual assets/shares and integer rounding, so the conversion is not linear once rewards make `totalAssets()` differ from `totalSupply()`.

In reachable post-reward states, `maxMint()` consequently overstates the executable capacity by one raw share. An unprivileged user who queries `maxMint()` and passes that exact advertised amount to `mint()` receives `DepositLimitExceeded`, even though ERC-4626 requires `maxMint()` never to exceed the amount that `mint()` will accept without reverting. Two attached local PoCs execute the real legacy-to-V2 upgrade, the real `VaultRewardDistributor.distributeIncentive()` path, the real `maxMint()` calculation, and the real failing `mint()` transaction. The primary PoC replays the exact publicly observable Phase-1 supply, assets, and deposit limit, then reaches the failing residue through a `0.001054 FXRP` permissionless deposit. A one-share-lower negative control succeeds.

### Affected asset and frozen snapshot

The selected in-scope asset is:

`https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol`

The report and PoC are pinned to:

```
42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

Relevant frozen code:

* [`FirelightVault.maxMint()` lines 385-392](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L385-L392)
* [`FirelightVault.mint()` lines 682-706](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L682-L706)
* [`FirelightVault._depositFunds()` lines 976-989](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L989)
* [`VaultRewardDistributor.distributeIncentive()` lines 163-176](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L163-L176)

### Details

The vulnerable implementation is:

```solidity
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

Let:

* `S = totalSupply()`;
* `A = totalAssets()`;
* `L = depositLimit`; and
* `R = L - A`, the actual remaining asset capacity.

The inherited OpenZeppelin conversion uses a zero decimals offset, one virtual share, and one virtual asset. Therefore:

```
convertToShares(x) = floor(x * (S + 1) / (A + 1))
previewMint(k)     = ceil(k * (A + 1) / (S + 1))
```

The current function returns:

```
reportedMax = floor(L * (S + 1) / (A + 1)) - S
```

However, `mint(k)` is executable under the deposit limit only when:

```
ceil(k * (A + 1) / (S + 1)) <= L - A
```

Thus, the actual maximum is:

```
executableMax = floor((L - A) * (S + 1) / (A + 1))
```

Converting the absolute cap and then subtracting supply is not equivalent to converting the remaining asset capacity. When rewards make `A > S`, the extra rounding term can make `reportedMax = executableMax + 1`.

The PoC reaches the following state through ordinary contract calls:

```
S = 1,000,000 raw shares
A = 1,050,000 raw FXRP after a 5% incentive
L = 2,050,000 raw FXRP
R = 1,000,000 raw FXRP

maxMint()                         = 952,381 shares
previewDeposit(maxDeposit())      = 952,380 shares
previewMint(maxMint())            = 1,000,001 assets
remaining asset capacity          = 1,000,000 assets
```

`mint(952,381)` therefore reaches `_depositFunds()`, calculates total assets one unit above `depositLimit`, and reverts with `DepositLimitExceeded`. In the same state, `mint(952,380)` succeeds and emits `Deposit`.

#### Current deployed Phase-1 state replay

The stronger PoC pins the following read-only state from the deployed Phase-1 proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` at Flare block `68,068,112` (`2026-08-23T10:45:27Z`):

```
totalSupply  = 56,388,448,921,177 raw shares
totalAssets  = 56,392,505,684,922 raw FXRP
depositLimit = 60,168,000,000,000 raw FXRP
paused       = false
```

The local replay reaches that exact assets/supply ratio through the real `VaultRewardDistributor` after the intended V2 upgrade. Before the trigger, `previewMint(maxMint) == maxDeposit`, providing a safe-state negative control. An ordinary user then deposits only `1,054` raw FXRP (`0.001054 FXRP`) through the public `deposit()` function. It mints `1,053` raw shares and reaches:

```
totalSupply                  = 56,388,448,922,230
totalAssets                  = 56,392,505,685,976
remaining asset capacity     = 3,775,494,314,024
maxMint()                    = 3,775,222,712,536
actual executable maximum    = 3,775,222,712,535
previewMint(maxMint())       = 3,775,494,314,025
```

The advertised amount therefore exceeds the remaining asset capacity by one raw unit and its real `mint()` call reverts. The next-lower mint succeeds. This replay uses no time warp, storage write, direct vault donation, impersonation, or attacker-controlled privileged call. The pinned Phase-1 values are public-state evidence only; every state transition in the PoC occurs on a fresh local chain.

### Trigger flow

1. The intended upgrade retains the deployed Phase-1 vault's supply, asset balance, and deposit limit in the frozen V2 `FirelightVault` implementation.
2. A normal user deposits `1,054` raw FXRP through public `deposit()`, receiving `1,053` raw shares. No privileged action is part of this trigger.
3. A second unprivileged user calls `maxMint(receiver)` and receives `3,775,222,712,536` shares.
4. `previewMint(maxMint)` requires `3,775,494,314,025` assets, while `maxDeposit` and the real remaining capacity are only `3,775,494,314,024`.
5. The user calls `mint(maxMint, receiver)`. The production `mint()` and `_depositFunds()` path reverts with `DepositLimitExceeded`.
6. As a negative control, `mint(3,775,222,712,535, receiver)` succeeds, proving that the failure is the one-share overstatement rather than pause, blocklist, incident, allowance, balance, upgrade, or setup failure.

The secondary PoC independently reaches the same root from a small clean state via an ordinary 5% incentive and reproduces `952,381` advertised versus `952,380` executable shares.

### Preconditions and feasibility

* The V2 upgrade has occurred.
* The asset/share exchange rate is above 1:1. The currently deployed Phase-1 vault already satisfies this condition at the pinned block.
* The vault remains below its configured deposit limit and reaches a rounding boundary. From the pinned deployed state, the primary PoC reaches it through a `1,054`-raw-unit public deposit without changing the configured limit.
* The receiver is not blocklisted, the vault is not paused, and there is no active incident.
* An unprivileged user follows the standard `maxMint()` -> `mint()` integration pattern.

The finding does **not** require a malicious or incorrect action by a privileged role holder. The triggering state transition is a permissionless deposit. The production reward distributor is used only to reproduce the already-observed assets/supply ratio on a fresh local chain. There is no race, stale off-chain state, direct storage mutation, malicious token behavior, or public-chain transaction.

### Why this is distinct from OpenZeppelin CR-01 and PR #9

The public OpenZeppelin report's CR-01 is titled **"Limit Functions Not Overridden to Enforce Deposit/Withdraw Restrictions."** Its factual root cause was that none of the four inherited ERC-4626 limit functions were overridden to account for the deposit limit, blocklist, or pause state. Its recommendation was to add overrides, and the report marks it resolved by PR #9 at commit `ddb821b`.

This report begins from the already-added `maxMint()` override and demonstrates a different defect inside the remediation: the new implementation converts the absolute cap before subtracting supply, which becomes incorrect after yield due to virtual offsets and rounding. Fixing this report requires replacing the new formula; merely adding the already-present override does not fix it.

The public [PR #9](https://github.com/firelight-protocol/firelight-core/pull/9) has no description, no review, and no discussion of this arithmetic condition or its impact. Its [single public commit](https://github.com/firelight-protocol/firelight-core/commit/ddb821be58928c32a10e7d85993a419c91b65872) contains the current formula and tests `maxMint == previewDeposit(maxDeposit)` only while assets and shares remain at a 1:1 exchange rate. Those tests pass and do not disclose:

* the post-reward `A > S` condition;
* the one-raw-share overstatement;
* `previewMint(maxMint) > maxDeposit`; or
* the resulting real `mint(maxMint)` revert.

Accordingly, the cited audit and PR disclose the missing-override issue and its attempted remediation, not the ineffective-fix behavior demonstrated here. Immunefi's Firelight Code Freeze Assurance states that a bypass of a fix is a new, valid bug. Immunefi's general guidance likewise states that a previously found bug whose fix did not cover all exploitation means, or a new bug introduced by a fix, should be reported if the resulting impact is in scope.

## Impact

The selected impact is:

> **Low — Contract fails to deliver promised returns, but doesn't lose value**

ERC-4626 specifies that `maxMint()` must return the maximum number of shares that `mint()` will accept for the receiver without reverting, and it must not return a higher amount; underestimation is explicitly preferred when necessary. Here the vault advertises a maximum that its own production `mint()` rejects in the same state.

This can break wallets, routers, vault aggregators, or user scripts that use the standard boundary pattern `shares = maxMint(receiver); mint(shares, receiver)`. The official Flare Firelight mint guide itself treats `maxMint()` as the capacity validation step before executing `mint()`.

The impact is immediate and reproducible: the boundary mint reverts and the caller pays for a failed transaction. This report does **not** claim theft, insolvency, permanent or temporary freezing, loss of principal, or attacker profit. The overstatement is at most one raw share for this root cause, and retrying with the correct one-share-lower amount succeeds. Those limitations are why Low, rather than Medium or higher, is selected.

## Risk Breakdown

* **Severity:** Low
* **In-scope impact:** Contract fails to deliver promised returns, but doesn't lose value
* **Affected caller:** Any unprivileged minter using the advertised boundary
* **Required protocol state:** Positive reward/yield, remaining deposit headroom, and a reachable rounding boundary
* **Privileges required by the triggering caller:** None
* **User interaction:** The user queries `maxMint()` and sends the corresponding `mint()` transaction
* **Repeatability:** The failure is deterministic for the demonstrated state until totals or the deposit limit change
* **Funds at risk:** None claimed
* **Recovery:** The user can retry with a smaller amount; no funds are locked

## Recommendation

Convert the **remaining asset capacity** to shares instead of converting the absolute cap and subtracting the existing supply. For example:

```solidity
function maxMint(address receiver) public view override returns (uint256 amount) {
    uint256 assets = totalAssets();

    if (
        isBlocklisted[receiver] ||
        paused() ||
        _hasActiveIncident() ||
        assets >= depositLimit
    ) {
        return 0;
    }

    return previewDeposit(depositLimit - assets);
}
```

Equivalent reuse of `previewDeposit(maxDeposit(receiver))` is also appropriate as long as all global and receiver-specific guards remain represented.

Add regression/invariant coverage for exchange rates both above and below 1:1, including ordinary rewards:

```
previewMint(maxMint(receiver)) <= maxDeposit(receiver)
```

For a finite nonzero maximum, also verify that the next raw share is not accepted under the same limit:

```
previewMint(maxMint(receiver) + 1) > maxDeposit(receiver)
```

## References

* [EIP-4626 `maxMint` requirements](https://eips.ethereum.org/EIPS/eip-4626#maxmint)
* [Frozen `FirelightVault.maxMint()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L385-L392)
* [Frozen `FirelightVault.mint()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L682-L706)
* [Frozen deposit-limit enforcement](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L989)
* [Frozen reward/incentive production path](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L132-L176)
* [OpenZeppelin Firelight audit containing CR-01](https://github.com/firelight-protocol/firelight-core/blob/main/audits/2025-07-21_OpenZeppelin_Firelight.pdf)
* [Public PR #9](https://github.com/firelight-protocol/firelight-core/pull/9)
* [PR #9 commit `ddb821b`](https://github.com/firelight-protocol/firelight-core/commit/ddb821be58928c32a10e7d85993a419c91b65872)
* [Firelight competition scope and Code Freeze Assurance](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/)
* [Immunefi guidance on bypassing an ineffective fix](https://immunefisupport.zendesk.com/hc/en-us/articles/18517917674641-What-if-my-bug-report-can-bypass-a-fix)
* [Immunefi known-issue proof standard](https://immunefisupport.zendesk.com/hc/en-us/articles/10644746170897-Report-Closed-for-Known-Issues)
* [Official Flare Firelight mint guide](https://dev.flare.network/fxrp/firelight/mint)

### Proof of Concept

## Proof of Concept

### What the PoC executes

The two PoCs are end-to-end on a fresh local Hardhat chain. They do not replace any security-relevant transition with a local variable or arithmetic-only model. Together, they:

1. deploy the repository's local FXRP test token;
2. deploys the actual `FirelightVaultPredeposits` implementation behind an upgradeable proxy;
3. executes a real initial deposit;
4. upgrades that proxy to the frozen `FirelightVault` implementation;
5. executes `initializeV2()` and verifies `contractVersion() == 2`;
6. deploys the actual frozen `VaultRewardDistributor` behind a proxy;
7. grants it the vault's real `CHECKPOINT_ROLE`;
8. executes a real `distributeIncentive()` token transfer and checkpoint;
9. reads the resulting real `totalSupply()`, `totalAssets()`, `maxDeposit()`, `maxMint()`, `previewDeposit()`, and `previewMint()` values;
10. in the primary replay, executes the public `1,054`-raw-unit deposit that moves the pinned state from a safe residue to the failing residue;
11. sends the advertised `mint(maxMint)` transaction and asserts the real `DepositLimitExceeded` revert; and
12. sends the corrected negative-control mint and asserts a real `Deposit` event and final assets not exceeding the cap.

No public testnet or mainnet is contacted.

### Attachment and placement

Attach both supplied files to the report. After checking out the frozen repository, place them at:

```
test/security-max-mint-live-phase1-state.js
test/security-max-mint-upgrade-path.js
```

Attachment SHA-256 values:

```
security-max-mint-live-phase1-state.js
b27ecc95c2deb1b64a945d2d16d3879f1ce120b561bd0249946bb3cad385c4c1

security-max-mint-upgrade-path.js
812ab52118a554ec2f8b50b780583132a2dda9a8ceb884c32f58c3b7ab3eb4ec
```

The secondary attachment is reproduced below. The primary attachment is included separately because it adds the pinned deployed-state replay and pre-trigger control.

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const { deployFAsset } = require('../lib/utils_test')

describe('PoC (upgrade path): maxMint is not executable after ordinary yield', function () {
  const INITIAL_DEPOSIT = 1_000_000n // 1 FXRP
  const REWARD = 50_000n // 0.05 FXRP (5% yield)
  const DEPOSIT_LIMIT = 2_050_000n // 2.05 FXRP
  const PERIOD_DURATION = 604_800

  it('survives the legacy-to-V2 upgrade and makes mint(maxMint) revert', async function () {
    const [admin, distributor, user0, user1] = await ethers.getSigners()
    const { token_contract: asset } = await deployFAsset(['fXRP', 'fXRP', 'Ripple', 'XRP', 6])

    const initParams = ethers.AbiCoder.defaultAbiCoder().encode(
      ['address', 'address', 'address', 'address', 'address', 'address', 'uint256', 'uint48'],
      [
        admin.address,
        admin.address,
        admin.address,
        admin.address,
        admin.address,
        admin.address,
        DEPOSIT_LIMIT,
        PERIOD_DURATION,
      ]
    )

    const LegacyVault = await ethers.getContractFactory('FirelightVaultPredeposits')
    const legacyVault = await upgrades.deployProxy(LegacyVault, [
      await asset.getAddress(),
      'stfXRP',
      'stfXRP',
      initParams,
    ])

    await asset.mintTo(user0.address, INITIAL_DEPOSIT)
    await asset.connect(user0).approve(await legacyVault.getAddress(), INITIAL_DEPOSIT)
    await legacyVault.connect(user0).deposit(INITIAL_DEPOSIT, user0.address)
    expect(await legacyVault.contractVersion()).to.equal(1n)

    const V2Vault = await ethers.getContractFactory('FirelightVault')
    const vault = await upgrades.upgradeProxy(await legacyVault.getAddress(), V2Vault)
    await vault.initializeV2()
    expect(await vault.contractVersion()).to.equal(2n)

    const RewardDistributor = await ethers.getContractFactory('VaultRewardDistributor')
    const rewardDistributor = await upgrades.deployProxy(RewardDistributor, [
      await vault.getAddress(),
      admin.address,
      distributor.address,
      ethers.ZeroAddress,
    ])
    await vault.grantRole(await vault.CHECKPOINT_ROLE(), await rewardDistributor.getAddress())

    await asset.mintTo(distributor.address, REWARD)
    await asset.connect(distributor).approve(await rewardDistributor.getAddress(), REWARD)
    await rewardDistributor
      .connect(distributor)
      .distributeIncentive(REWARD, ethers.id('ordinary-five-percent-incentive'))

    await asset.mintTo(user1.address, DEPOSIT_LIMIT)
    await asset.connect(user1).approve(await vault.getAddress(), DEPOSIT_LIMIT)

    expect(await vault.totalSupply()).to.equal(INITIAL_DEPOSIT)
    expect(await vault.totalAssets()).to.equal(INITIAL_DEPOSIT + REWARD)

    const remainingAssets = await vault.maxDeposit(user1.address)
    const advertisedMax = await vault.maxMint(user1.address)
    const executableMax = await vault.previewDeposit(remainingAssets)

    expect(remainingAssets).to.equal(1_000_000n)
    expect(advertisedMax).to.equal(952_381n)
    expect(executableMax).to.equal(952_380n)
    expect(advertisedMax).to.equal(executableMax + 1n)
    expect(await vault.previewMint(advertisedMax)).to.equal(remainingAssets + 1n)
    const executableAssets = await vault.previewMint(executableMax)
    expect(executableAssets).to.equal(remainingAssets - 1n)

    await expect(
      vault.connect(user1).mint(advertisedMax, user1.address)
    ).to.be.revertedWithCustomError(vault, 'DepositLimitExceeded')

    await expect(
      vault.connect(user1).mint(executableMax, user1.address)
    ).to.emit(vault, 'Deposit')
    expect(await vault.totalAssets()).to.equal(INITIAL_DEPOSIT + REWARD + executableAssets)
    expect(await vault.totalAssets()).to.be.at.most(DEPOSIT_LIMIT)
  })
})
```

### Reproduction commands

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install

# Copy both attachments into test/, then run:
npx hardhat test test/security-max-mint-live-phase1-state.js test/security-max-mint-upgrade-path.js
```

### Observed output

```
PoC (pinned Phase-1 state): an ordinary deposit makes maxMint non-executable
  √ preserves the live snapshot through upgrade, then mint(maxMint) reverts

PoC (upgrade path): maxMint is not executable after ordinary yield
  √ survives the legacy-to-V2 upgrade and makes mint(maxMint) revert

2 passing
```

### Security assertions demonstrated

* The state is reached through the intended legacy-to-V2 deployment path.
* The primary replay starts from the exact deployed Phase-1 `S/A/L` values and proves the current residue is safe before an ordinary deposit reaches the bug.
* A real, ordinary 5% incentive changes the exchange rate; no direct donation shortcut is used.
* No time warp, storage write, direct vault donation, or impersonation is used.
* `maxMint()` advertises `952,381` shares.
* The advertised amount requires `1,000,001` assets against `1,000,000` of remaining capacity.
* The real production `mint(maxMint)` transaction reverts with `DepositLimitExceeded`.
* The negative-control `mint(952,380)` transaction succeeds and leaves `totalAssets() <= depositLimit`.
