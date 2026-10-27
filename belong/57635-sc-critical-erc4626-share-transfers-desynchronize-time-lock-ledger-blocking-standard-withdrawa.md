# 57635 sc critical erc4626 share transfers desynchronize time lock ledger blocking standard withdrawals for recipients

**Submitted on Oct 27th 2025 at 19:29:38 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57635
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Issue description

The `Staking` vault inherits Solady `ERC4626` (transferable by default) and implements a time-lock by keeping a per-user ledger:

```solidity
/// @notice User stake entries stored as arrays per staker.
/// @dev Public getter: stakes(user, i) → (shares, timestamp).
mapping(address staker => Stake[] times) public stakes;
```

Lock entries are created only on deposit (mint):

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

Withdrawals require consuming unlocked shares from `stakes[_owner]` and will revert if not enough unlocked shares exist:

```solidity
function _withdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal override {
    _consumeUnlockedSharesOrRevert(_owner, shares);
    super._withdraw(by, to, _owner, assets, shares);
}
```

The unlocked-only consumption routine:

```solidity
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 _min = minStakePeriod;
    uint256 nowTs = block.timestamp;
    uint256 remaining = need;

    for (uint256 i; i < userStakes.length && remaining > 0;) {
        Stake memory s = userStakes[i];
        if (nowTs >= s.timestamp + _min) {
            uint256 take = s.shares <= remaining ? s.shares : remaining;
            if (take == s.shares) {
                remaining -= take;
                userStakes[i] = userStakes[userStakes.length - 1];
                userStakes.pop();
            } else {
                userStakes[i].shares = s.shares - take;
                remaining = 0;
                ++i;
            }
        } else {
            ++i;
        }
    }

    if (remaining != 0) revert MinStakePeriodNotMet();
}
```

Solady’s ERC20 implementation calls transfer hooks (`_beforeTokenTransfer` / `_afterTokenTransfer`) that are available to override, but `Staking` does not override them to maintain `stakes` consistency when shares are transferred:

```solidity
function transfer(address to, uint256 amount) public virtual returns (bool) {
    _beforeTokenTransfer(msg.sender, to, amount);
    // ... move balances ...
    _afterTokenTransfer(msg.sender, to, amount);
    return true;
}
```

Because `stakes` is not mirrored on transfers, transferring shares increases the recipient’s ERC4626 balance but leaves `stakes[recipient]` empty. As a result, standard `withdraw`/`redeem` calls revert for recipients with transferred shares (they have shares but no unlocked stake entries). Recipients must either re-transfer the shares back or use the emergency redemption path (which imposes a penalty).

### Impact

Medium — Griefing

* Users receiving transferred shares cannot use standard `withdraw`/`redeem`. They must either re-transfer the shares (inconvenience and possible extra fees) or use `emergencyRedeem` which applies a penalty.

## Recommended mitigation steps

Two main approaches:

* Restrict transfers:
  * Override `ERC20._beforeTokenTransfer` in `Staking` to revert when `from != address(0)` and `to != address(0)` (i.e., disallow transfers between accounts while preserving mint/burn).
* Mirror locks on transfers:
  * Override `ERC20._beforeTokenTransfer` or `_afterTokenTransfer` to handle `from != address(0) && to != address(0)`:
    * Debit `amount` from `stakes[from]` using oldest-first order, splitting entries as needed.
    * Push equivalent entries to `stakes[to]`, preserving original timestamps.
    * Ensure mints/burns are handled so `_deposit` does not double-record entries.
  * Deterministic ordering prevents “lock bleaching.”

Do not add behavior changes beyond mirroring or transfer restrictions. Preserve timestamps when mirroring to avoid bypassing min-stake periods.

## Proof of Concept

The following PoC demonstrates the issue: transferred shares appear in ERC4626 views (e.g., `maxRedeem`) but runtime lock checks revert `redeem`/`withdraw` for the recipient because their `stakes` array is empty. `emergencyRedeem` still succeeds (but with a penalty).

Use the provided test & config snippets below to reproduce locally.

{% stepper %}
{% step %}

### PoC test file

Create `test/v2/periphery/staking.transfer-lock.poc.test.ts` with the following content:

```ts
import { expect } from 'chai';
import { ethers, upgrades } from 'hardhat';

describe('Staking transfer lock desync PoC', function () {
  it('Transferred shares cannot redeem/withdraw; emergencyRedeem succeeds', async function () {
    const [owner, alice, bob, treasury] = await ethers.getSigners();

    // Deploy LONG mock (18 decimals)
    const Erc20 = await ethers.getContractFactory('WETHMock');
    const long = await Erc20.deploy();
    await long.deployed();

    // Deploy Staking as a transparent proxy and initialize
    const Staking = await ethers.getContractFactory('Staking');
    const staking = await upgrades.deployProxy(
      Staking,
      [owner.address, treasury.address, long.address],
      { initializer: 'initialize', unsafeAllow: ['constructor'] }
    );
    await staking.deployed();

    // Alice mints LONG and deposits into the vault
    const depositAmount = ethers.utils.parseEther('100');
    await long.connect(alice).mint(alice.address, depositAmount);
    await long.connect(alice).approve(staking.address, ethers.constants.MaxUint256);

    const sharesMinted = await staking.connect(alice).callStatic.deposit(depositAmount, alice.address);
    await staking.connect(alice).deposit(depositAmount, alice.address);

    // Transfer half the shares to Bob
    const halfShares = sharesMinted.div(2);
    await staking.connect(alice).transfer(bob.address, halfShares);

    // Views show Bob can redeem, but runtime lock check will revert because stakes[bob] is empty
    const bobMaxRedeem = await staking.maxRedeem(bob.address);
    expect(bobMaxRedeem).to.equal(halfShares);

    await expect(
      staking.connect(bob).redeem(halfShares, bob.address, bob.address)
    ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');

    const assetsForHalf = await staking.previewRedeem(halfShares);
    await expect(
      staking.connect(bob).withdraw(assetsForHalf, bob.address, bob.address)
    ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');

    // Emergency path succeeds and pays out assets minus penalty
    const bobAssetsBefore = await long.balanceOf(bob.address);
    await staking.connect(bob).emergencyRedeem(halfShares, bob.address, bob.address);
    const bobAssetsAfter = await long.balanceOf(bob.address);
    expect(bobAssetsAfter).to.be.gt(bobAssetsBefore);
  });
});
```

{% endstep %}

{% step %}

### Hardhat config adjustments (optional)

Because of compatibility issues, the following changes were made to `hardhat.config.ts`. This file is provided for context — keep links and settings unchanged if you reuse it.

```ts
import { HardhatUserConfig } from 'hardhat/config';
import '@nomicfoundation/hardhat-toolbox';
import '@openzeppelin/hardhat-upgrades';
import 'solidity-docgen';
import 'hardhat-contract-sizer';
import '@nomicfoundation/hardhat-ledger';
import dotenv from 'dotenv';
import { ChainIds } from './utils/chain-ids';
import { blockscanConfig, createConnect, createLedgerConnect } from './utils/config';

dotenv.config();

let accounts: string[] = [],
  ledgerAccounts: string[] = [];

if (process.env.PK) {
  accounts = [process.env.PK];
}
if (process.env.LEDGER_ADDRESS) {
  ledgerAccounts = [process.env.LEDGER_ADDRESS];
}

// Allow disabling mainnet forking for local tests (e.g. NO_FORK=1)
// Optionally allow overriding fork URL via FORK_URL
const useForking =
  process.env.NO_FORK !== '1' && !!(process.env.INFURA_ID_PROJECT || process.env.FORK_URL);

const config: HardhatUserConfig = {
  solidity: {
    compilers: [
      {
        version: '0.8.27',
        settings: {
          optimizer: {
            enabled: true,
            runs: 200,
          },
        },
      },
    ],
  },
  networks: {
    hardhat: {
      ...(useForking
        ? {
            forking: {
              url:
                process.env.FORK_URL ||
                (process.env.INFURA_ID_PROJECT
                  ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
                  : `https://eth.llamarpc.com`),
              blockNumber: 23490636,
            },
          }
        : {}),
      // throwOnCallFailures: false,
      accounts: { accountsBalance: '10000000000000000000000000' },
      initialBaseFeePerGas: 0,
      allowUnlimitedContractSize: false,
    },
    // 'ethereum': {
    //   url: 'https://eth.drpc.org',
    // },
    mainnet: createLedgerConnect(ChainIds.mainnet, ledgerAccounts),
    bsc: createLedgerConnect(ChainIds.bsc, ledgerAccounts),
    polygon: createLedgerConnect(ChainIds.polygon, ledgerAccounts),
    blast: createLedgerConnect(ChainIds.blast, ledgerAccounts),
    celo: createLedgerConnect(ChainIds.celo, ledgerAccounts),
    base: createLedgerConnect(ChainIds.base, ledgerAccounts),
    linea: createLedgerConnect(ChainIds.linea, ledgerAccounts),
    astar: createLedgerConnect(ChainIds.astar, ledgerAccounts),
    arbitrum: createLedgerConnect(ChainIds.arbitrum, ledgerAccounts),
    skale_europa: createLedgerConnect(ChainIds.skale_europa, ledgerAccounts),
    skale_nebula: createLedgerConnect(ChainIds.skale_nebula, ledgerAccounts),
    skale_calypso: createLedgerConnect(ChainIds.skale_calypso, ledgerAccounts),
    sepolia: createConnect(ChainIds.sepolia, accounts),
    blast_sepolia: createConnect(ChainIds.blast_sepolia, accounts),
    skale_calypso_testnet: createConnect(ChainIds.skale_calypso_testnet, accounts),
    amoy: createConnect(ChainIds.amoy, accounts),
  },
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY! || '',
      // 'ethereum': 'empty',
      blast: process.env.BLASTSCAN_API_KEY! || '',
      polygon: process.env.POLYSCAN_API_KEY || '',
      celo: process.env.CELOSCAN_API_KEY || '',
      base: process.env.BASESCAN_API_KEY || '',
      linea: process.env.LINEASCAN_API_KEY || '',
      sepolia: process.env.ETHERSCAN_API_KEY! || '',
      amoy: process.env.POLYSCAN_API_KEY || '',
      blast_sepolia: process.env.BLASTSCAN_API_KEY! || '',
      astar: 'astar', // Is not required by blockscout. Can be any non-empty string
      skale_europa: 'skale_europa', // Is not required by blockscout. Can be any non-empty string
      skale_nebula: 'skale_nebula', // Is not required by blockscout. Can be any non-empty string
      skale_calypso: 'skale_calypso', // Is not required by blockscout. Can be any non-empty string
      skale_calypso_testnet: 'skale_calypso_testnet', // Is not required by blockscout. Can be any non-empty string
    },
    customChains: [
      // {
      //   network: "ethereum",
      //   chainId: 1,
      //   urls: {
      //     apiURL: "https://eth.blockscout.com/api",
      //     browserURL: "https://eth.blockscout.com"
      //   }
      // },
      blockscanConfig('blast', ChainIds.blast),
      blockscanConfig('blast_sepolia', ChainIds.blast_sepolia),
      blockscanConfig('celo', ChainIds.celo),
      blockscanConfig('base', ChainIds.base),
      blockscanConfig('linea', ChainIds.linea),
      blockscanConfig('astar', ChainIds.astar),
      blockscanConfig('skale_europa', ChainIds.skale_europa),
      blockscanConfig('skale_nebula', ChainIds.skale_nebula),
      blockscanConfig('skale_calypso', ChainIds.skale_calypso),
      blockscanConfig('blast_sepolia', ChainIds.blast_sepolia),
      blockscanConfig('amoy', ChainIds.amoy),
      blockscanConfig('skale_calypso_testnet', ChainIds.skale_calypso_testnet),
    ],
  },
  paths: {
    sources: 'contracts',
  },
  docgen: {
    outputDir: './docs/contracts',
    exclude: ['nft-with-royalties/mocks', 'mocks'],
    pages: 'files',
  },
  gasReporter: {
    enabled: process.env.REPORT_GAS !== undefined,
    currency: 'USD',
  },
  mocha: {
    timeout: 180000, // defense in depth
    parallel: false, // parallel + fork tends to hang
  },
};

export default config;
```

{% endstep %}

{% step %}

### How to run the PoC

Run the test with:

```
NO_FORK=1 LEDGER_ADDRESS=0x0000000000000000000000000000000000000000 PK=0x0123456789012345678901234567890123456789012345678901234567890123 npx hardhat test test/v2/periphery/staking.transfer-lock.poc.test.ts
```

Expected test result:

```
   ✔ Transferred shares cannot redeem/withdraw; emergencyRedeem succeeds (439ms)


  1 passing (441ms)
```

{% endstep %}
{% endstepper %}

## Additional notes

* Do not modify external links or URLs from the original report.
* Two mitigation patterns are presented; choose the one that matches product requirements:
  * Disallow transfers if transfers are not intended.
  * Mirror stake entries on transfer if token transferability must be preserved.

If you want, I can generate a suggested patch for `Staking.sol` implementing either (A) a transfer restriction via `_beforeTokenTransfer` that reverts on non-mint/burn transfers, or (B) an implementation that mirrors stakes on transfers (debiting from sender's stakes oldest-first and crediting the recipient with preserved timestamps). Which option would you like to see implemented as a code patch?
