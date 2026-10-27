# #38015 \[BC-Insight] Violation of EIP-2681 in Create Transaction

**Submitted on Dec 21st 2024 at 20:30:28 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38015
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/NethermindEth/nethermind>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

EIP-2681 (<https://eips.ethereum.org/EIPS/eip-2681>) requires every Ethereum execution client to restrict the nonce of an account to 2^64 - 1. Specifically, a transaction with nonce 2^64 - 1 should fail, including create transaction and call transaction.\
In the current implementation of Nethermind, the threshold of nonce in the creation transaction is 2^64 - 2 which is 1 less than the threshold ( 2^64 - 1 ) specified in EIP-2681. This could be problematic once an account reaches this threshold, and potentially leading to consensus issues.

## Vulnerability Details

Affected Codebase:\
<https://github.com/NethermindEth/nethermind/tree/1.30.1>

The stateless validation of a tx is performed in the function `ValidateStatic()`, which contain the nonce check to ensure it does not exceed the threshold 2^64-1 specified in the\
EIP-2681 (<https://eips.ethereum.org/EIPS/eip-2681>)

<https://github.com/NethermindEth/nethermind/blob/1.30.1/src/Nethermind/Nethermind.Evm/TransactionProcessing/TransactionProcessor.cs#L348>

```
       protected virtual TransactionResult ValidateStatic(
            Transaction tx,
            BlockHeader header,
            IReleaseSpec spec,
            ExecutionOptions opts,
            out long intrinsicGas)
        {
            intrinsicGas = IntrinsicGasCalculator.Calculate(tx, spec);


            bool validate = !opts.HasFlag(ExecutionOptions.NoValidation);


            if (tx.SenderAddress is null)
            {
                TraceLogInvalidTx(tx, "SENDER_NOT_SPECIFIED");
                return TransactionResult.SenderNotSpecified;
            }


            if (validate && tx.Nonce >= ulong.MaxValue - 1)
            {
                // we are here if nonce is at least (ulong.MaxValue - 1). If tx is contract creation,
                // it is max possible value. Otherwise, (ulong.MaxValue - 1) is allowed, but ulong.MaxValue not.
                if (tx.IsContractCreation || tx.Nonce == ulong.MaxValue)
                {
                    TraceLogInvalidTx(tx, "NONCE_OVERFLOW");
                    return TransactionResult.NonceOverflow;
                }
            }

...
```

However, the nonce validation checks that the nonce for a create transaction does not exceed 2^64 - 2 while the nonce for a call transaction does not exceed 2^64 - 1. According to the EIP-2681 (<https://eips.ethereum.org/EIPS/eip-2681>), both nonce checks should be performed against 2 ^ 64 -1.

Since other Ethereum clients implementation aligns with the EIP-2681, the Nethermind client will fail out of consensus once an account reaches 2^64 - 2 and it invokes a create transaction.\
In this case, this transaction fails in the Nethermind client but succeeds in other Ethereum clients.

## Impact Details

Due to the violation of EIP-2681 (<https://eips.ethereum.org/EIPS/eip-2681>), it would lead to consensus issues once the nonce of 2^64 -2 is reached in an account and it invokes a create transaction.

## References

* <https://github.com/NethermindEth/nethermind/tree/1.30.1>
* EIP-2681 (<https://eips.ethereum.org/EIPS/eip-2681>)

## Proof of Concept

## Proof of Concept

1. For simplicity, we add the following test case in the test file:

nethermind/src/Nethermind/Nethermind.Evm.Test/TransactionProcessorTests.cs

where the nonce is set to ulong.MaxValue - 1 ( 2^64 - 1 ) for a create transaction, and it asserts the transaction will fail.

```
   [TestCase]
    public void Can_Not_execute_create_transaction_with_nonce()
    {


        PrivateKey sender = TestItem.PrivateKeyA;


        _stateProvider.CreateAccount(sender.Address, 1.Ether(), ulong.MaxValue-1);


        byte[] initByteCode = Prepare.EvmCode
            .ForInitOf(Bytes.FromHexString("6000")).Done;


        Address contractAddress = ContractAddress.From(TestItem.PrivateKeyA.Address, 0);


        byte[] byteCode = Prepare.EvmCode
            .Call(contractAddress, 46179).Done;


        long gasLimit = 100000;


        Transaction initTx = Build.A.Transaction.SignedAndResolved(_ethereumEcdsa, sender, _isEip155Enabled).WithCode(initByteCode).WithGasLimit(gasLimit).WithNonce(ulong.MaxValue-1).TestObject;
       
        Block block = Build.A.Block.WithNumber(MainnetSpecProvider.MuirGlacierBlockNumber).WithTransactions(initTx).WithGasLimit(2 * gasLimit).TestObject;


        TransactionResult result = Execute(initTx, block);
        Assert.That(result.Fail, Is.True);
}
```

2. Run the test case

```
dotnet test Nethermind.Evm.Test.csproj  --filter "FullyQualifiedName~Can_Not_execute_create_transaction_with_nonce"
```

3. The test result shows the create transaction fails

```
Restore complete (1.9s)
  Nethermind.Logging succeeded (0.7s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Logging/debug/Nethermind.Logging.dll
  Nethermind.Core succeeded (0.8s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Core/debug/Nethermind.Core.dll
  Nethermind.Serialization.Rlp succeeded (1.2s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Serialization.Rlp/debug/Nethermind.Serialization.Rlp.dll
  Nethermind.Serialization.Json succeeded (1.0s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Serialization.Json/debug/Nethermind.Serialization.Json.dll
  Nethermind.Abi succeeded (1.1s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Abi/debug/Nethermind.Abi.dll
  Nethermind.Config succeeded (1.3s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Config/debug/Nethermind.Config.dll
  Nethermind.Network.Stats succeeded (0.5s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Network.Stats/debug/Nethermind.Network.Stats.dll
  Nethermind.Db succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Db/debug/Nethermind.Db.dll
  Nethermind.Monitoring succeeded (0.7s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Monitoring/debug/Nethermind.Monitoring.dll
  Nethermind.Grpc succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Grpc/debug/Nethermind.Grpc.dll
  Nethermind.Sockets succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Sockets/debug/Nethermind.Sockets.dll
  Nethermind.Specs succeeded (0.8s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Specs/debug/Nethermind.Specs.dll
  Nethermind.Crypto succeeded (1.4s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Crypto/debug/Nethermind.Crypto.dll
  Nethermind.Network.Contract succeeded (0.5s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Network.Contract/debug/Nethermind.Network.Contract.dll
  Nethermind.Trie succeeded (0.9s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Trie/debug/Nethermind.Trie.dll
  Nethermind.KeyStore succeeded (0.3s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.KeyStore/debug/Nethermind.KeyStore.dll
  Nethermind.State succeeded (0.3s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.State/debug/Nethermind.State.dll
  Nethermind.Evm succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Evm/debug/Nethermind.Evm.dll
  Nethermind.TxPool succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.TxPool/debug/Nethermind.TxPool.dll
  Nethermind.Wallet succeeded (0.5s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Wallet/debug/Nethermind.Wallet.dll
  Nethermind.Blockchain succeeded (0.8s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Blockchain/debug/Nethermind.Blockchain.dll
  Nethermind.Consensus succeeded (0.5s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Consensus/debug/Nethermind.Consensus.dll
  Nethermind.Synchronization succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Synchronization/debug/Nethermind.Synchronization.dll
  Nethermind.Facade succeeded (0.7s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Facade/debug/Nethermind.Facade.dll
  Nethermind.Network succeeded (0.9s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Network/debug/Nethermind.Network.dll
  Nethermind.Network.Enr succeeded (0.3s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Network.Enr/debug/Nethermind.Network.Enr.dll
  Nethermind.Network.Dns succeeded (0.7s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Network.Dns/debug/Nethermind.Network.Dns.dll
  Nethermind.JsonRpc succeeded (1.0s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.JsonRpc/debug/Nethermind.JsonRpc.dll
  Nethermind.Db.Rpc succeeded (0.4s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Db.Rpc/debug/Nethermind.Db.Rpc.dll
  Nethermind.Api succeeded (0.3s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Api/debug/Nethermind.Api.dll
  Nethermind.Db.Rocks succeeded (0.7s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Db.Rocks/debug/Nethermind.Db.Rocks.dll
  Nethermind.Network.Discovery succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Network.Discovery/debug/Nethermind.Network.Discovery.dll
  Nethermind.Consensus.Ethash succeeded (1.0s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Consensus.Ethash/debug/Nethermind.Consensus.Ethash.dll
  Nethermind.Init succeeded (0.5s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Init/debug/Nethermind.Init.dll
  Nethermind.Consensus.AuRa succeeded (0.5s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Consensus.AuRa/debug/Nethermind.Consensus.AuRa.dll
  Nethermind.Specs.Test succeeded (0.6s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Specs.Test/debug/Nethermind.Specs.Test.dll
  Nethermind.Core.Test succeeded (0.8s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Core.Test/debug/Nethermind.Core.Test.dll
  Nethermind.Evm.Test succeeded (1.4s) → /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Evm.Test/debug/Nethermind.Evm.Test.dll
NUnit Adapter 4.6.0.0: Test execution started
Running selected tests in /Users/***/immunefi/nethermind/src/Nethermind/artifacts/bin/Nethermind.Evm.Test/debug/Nethermind.Evm.Test.dll
   NUnit3TestExecutor discovered 2 of 2 NUnit test cases using Current Discovery mode, Non-Explicit run
NUnit Adapter 4.6.0.0: Test execution complete
  Nethermind.Evm.Test test succeeded (3.2s)


Test summary: total: 2, failed: 0, succeeded: 2, skipped: 0, duration: 3.2s
Build succeeded in 21.3s
```
