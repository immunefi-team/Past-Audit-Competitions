# #38554 \[BC-Low] Incorrect Transaction Fee Check in \`SendRawTransaction()\`

**Submitted on Jan 6th 2025 at 16:02:18 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38554
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The `SendRawTransaction()` allows users to submit signed raw transactions to the local Erigon nodes, which contains a transaction fee check to ensure the transaction does not provide too many fees.

However, the transaction fees calculation is incorrect due to incorrect fetch of gas price for the transaction types, DynamicFeeTxType, BlobTxType and SetCodeTxType. Specifically, the gas price fetched from these types of transactions is the Tip instead of the FeeCap. This overlook makes this transaction fees ineffective as the fee cap is normally much larger than the tip.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/releases/tag/v3.0.0-alpha7>

The function `SendRawTransaction()` is intended to allow users to submit signed raw transactions to Erigon nodes:

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/turbo/jsonrpc/send\\_transaction.go#L18>

```
// SendRawTransaction implements eth_sendRawTransaction. Creates new message call transaction or a contract creation for previously-signed transactions.
func (api *APIImpl) SendRawTransaction(ctx context.Context, encodedTx hexutility.Bytes) (common.Hash, error) {
	txn, err := types.DecodeWrappedTransaction(encodedTx)
	if err != nil {
		return common.Hash{}, err
	}


	// If the transaction fee cap is already specified, ensure the
	// fee of the given transaction is _reasonable_.
	if err := checkTxFee(txn.GetPrice().ToBig(), txn.GetGas(), api.FeeCap); err != nil {
		return common.Hash{}, err
	}
	if !txn.Protected() && !api.AllowUnprotectedTxs {
		return common.Hash{}, errors.New("only replay-protected (EIP-155) transactions allowed over RPC")
	}


	// this has been moved to prior to adding of transactions to capture the
	// pre state of the db - which is used for logging in the messages below
	tx, err := api.db.BeginTemporalRo(ctx)
	if err != nil {
		return common.Hash{}, err
	}


	defer tx.Rollback()


	cc, err := api.chainConfig(ctx, tx)
	if err != nil {
		return common.Hash{}, err
	}


	if txn.Protected() {
		txnChainId := txn.GetChainID()
		chainId := cc.ChainID
		if chainId.Cmp(txnChainId.ToBig()) != 0 {
			return common.Hash{}, fmt.Errorf("invalid chain id, expected: %d got: %d", chainId, *txnChainId)
		}
	}


	hash := txn.Hash()
	res, err := api.txPool.Add(ctx, &txPoolProto.AddRequest{RlpTxs: [][]byte{encodedTx}})
	if err != nil {
		return common.Hash{}, err
	}


	if res.Imported[0] != txPoolProto.ImportResult_SUCCESS {
		return hash, fmt.Errorf("%s: %s", txPoolProto.ImportResult_name[int32(res.Imported[0])], res.Errors[0])
	}


	return txn.Hash(), nil
}
```

It first decodes the raw transaction and calls the function `checkTxFee()` to ensure the provided transaction fee does not exceed the pre-configured 1 Ether.

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/turbo/jsonrpc/send\\_transaction.go#L75>

```
// checkTxFee is an internal function used to check whether the fee of
// the given transaction is _reasonable_(under the cap).
func checkTxFee(gasPrice *big.Int, gas uint64, gasCap float64) error {
	// Short circuit if there is no gasCap for transaction fee at all.
	if gasCap == 0 {
		return nil
	}
	feeEth := new(big.Float).Quo(new(big.Float).SetInt(new(big.Int).Mul(gasPrice, new(big.Int).SetUint64(gas))), new(big.Float).SetInt(big.NewInt(params.Ether)))
	feeFloat, _ := feeEth.Float64()
	if feeFloat > gasCap {
		return fmt.Errorf("tx fee (%.2f ether) exceeds the configured cap (%.2f ether)", feeFloat, gasCap)
	}
	return nil
}
```

The issue is that the gas price fetched from the transaction is mistakenly set as the `tip` in the transaction types, `DynamicFeeTxType`, `BlobTxType` and `SetCodeTxType`. For example, if the transaction if of type `DynamicFeeTxType`:

<https://github.com/erigontech/erigon/blob/34714c0c25cc59587240ae7abc1c2758315254af/core/types/dynamic\\_fee\\_tx.go#L43C1-L43C76>

```
func (tx *DynamicFeeTransaction) GetPrice() *uint256.Int  { return tx.Tip }
```

The function `GetPrice()` returns the `Tip` in the transaction, which should be the `FeeCap`.\
In this case, `checkTxFee()` does not validate the transaction fees correctly, as the Tip in a transaction is much less than the `FeeCap`.

For example, even though the `FeeCap` exceeds the limit, the `Tip` does not. Consequently, the check does not fail.

## Impact Details

The incorrect fetch of gas price of transactions makes the transaction fee validation ineffective. For example, a transaction with a transaction fee exceeding the limit does not fail.

## References

* <https://github.com/erigontech/erigon/releases/tag/v3.0.0-alpha7>
* <https://ethereum.org/en/developers/docs/gas/>

## Proof of Concept

## Proof of Concept

For simplicity, we modified the test function `TestSendRawTransaction()` in the file: <https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/turbo/jsonrpc/send\\_transaction\\_test.go#L93>

1. Create a transaction of type DynamicFeeTxType and submit it:

```
func TestSendRawTransaction(t *testing.T) {
	mockSentry, require := mock.MockWithTxPool(t), require.New(t)
	logger := log.New()

	oneBlockStep(mockSentry, require, t)

	expectedValue := uint64(1234)
	//txn, err := types.SignTx(types.NewTransaction(0, common.Address{1}, uint256.NewInt(expectedValue), params.TxGas, uint256.NewInt(10*params.GWei), nil), *types.LatestSignerForChainID(mockSentry.ChainConfig.ChainID), mockSentry.Key)
	txn, err := types.SignTx(types.NewEIP1559Transaction(uint256.Int{1337}, 0, common.Address{1}, uint256.NewInt(expectedValue), 1_000_000, uint256.NewInt(2000_000), uint256.NewInt(3000_000), uint256.NewInt(4000_000), nil), *types.LatestSignerForChainID(mockSentry.ChainConfig.ChainID), mockSentry.Key)
	require.NoError(err)

	fmt.Printf("gas limit is %d \n", txn.GetGas())
	fmt.Printf("gas price is %d \n", txn.GetPrice())
	fmt.Printf("tip is %d \n", txn.GetTip())
	fmt.Printf("fee cap is %d \n", txn.GetFeeCap())

	fmt.Printf("gas price is equal to tip: %t \n", txn.GetPrice() == txn.GetTip())
	fmt.Printf("gas price is equal to fee cap: %t \n", txn.GetPrice() == txn.GetFeeCap())

	ctx, conn := rpcdaemontest.CreateTestGrpcConn(t, mockSentry)
	txPool := txpool.NewTxpoolClient(conn)
	ff := rpchelper.New(ctx, rpchelper.DefaultFiltersConfig, nil, txPool, txpool.NewMiningClient(conn), func() {}, mockSentry.Log)
	api := jsonrpc.NewEthAPI(newBaseApiForTest(mockSentry), mockSentry.DB, nil, txPool, nil, 5000000, 1e18, 100_000, false, 100_000, 128, logger)

	buf := bytes.NewBuffer(nil)
	err = txn.MarshalBinary(buf)
	require.NoError(err)

	txsCh, id := ff.SubscribePendingTxs(1)
	defer ff.UnsubscribePendingTxs(id)

	txHash, err := api.SendRawTransaction(ctx, buf.Bytes())
	require.NoError(err)

	select {
	case got := <-txsCh:
		require.Equal(expectedValue, got[0].GetValue().Uint64())
	case <-time.After(20 * time.Second): // Sometimes the channel times out on github actions
		t.Log("Timeout waiting for txn from channel")
		jsonTx, err := api.GetTransactionByHash(ctx, txHash)
		require.NoError(err)
		require.Equal(expectedValue, jsonTx.Value.Uint64())
	}

	//send same txn second time and expect error
	_, err = api.SendRawTransaction(ctx, buf.Bytes())
	require.NotNil(err)
	expectedErr := txpool_proto.ImportResult_name[int32(txpool_proto.ImportResult_ALREADY_EXISTS)] + ": " + txpoolcfg.AlreadyKnown.String()
	require.Equal(expectedErr, err.Error())
	mockSentry.ReceiveWg.Wait()

	//TODO: make propagation easy to test - now race
	//time.Sleep(time.Second)
	//sent := m.SentMessage(0)
	//require.Equal(eth.ToProto[m.MultiClient.Protocol()][eth.NewPooledTransactionHashesMsg], sent.Id)
}
```

2. The test result shows that the `GetPrice()` fetches the `Tip` instead of the `FeeCap`.

```
=== RUN   TestSendRawTransaction
gas limit is 1000000 
gas price is 3000000 
tip is 3000000 
fee cap is 4000000 
gas price is equal to tip: true 
gas price is equal to fee cap: false 
--- PASS: TestSendRawTransaction (0.18s)
PASS


Process finished with the exit code 0
```
