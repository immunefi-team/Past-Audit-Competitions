# 55524 bc insight null body transaction submission crashes rpc handler

**Submitted on Oct 1st 2025 at 16:59:22 UTC by @humanitia for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #55524
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/tree/release/hayabusa>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

### Brief / Intro

A `POST /transactions` with JSON body `null` causes the node panic, tearing down the HTTP handler and dropping the client connection. Any software relying on the RPC experiences outage.

### Vulnerability Details

Vulnerable code (<https://github.com/vechain/thor/blob/release/hayabusa/api/transactions/transactions.go#L129-L150>):

```go
func (t *Transactions) handleSendTransaction(w http.ResponseWriter, req *http.Request) error {
	var rawTx *api.RawTx
	if err := restutil.ParseJSON(req.Body, &rawTx); err != nil {
		return restutil.BadRequest(errors.WithMessage(err, "body"))
	}
	tx, err := rawTx.Decode()
	if err != nil {
		return restutil.BadRequest(errors.WithMessage(err, "raw"))
	}

	if err := t.pool.AddLocal(tx); err != nil {
		if txpool.IsBadTx(err) {
			return restutil.BadRequest(err)
		}
		if txpool.IsTxRejected(err) {
			return restutil.Forbidden(err)
		}
		return err
	}
	txID := tx.ID()
	return restutil.WriteJSON(w, &api.SendTxResult{ID: &txID})
}
```

When `ParseJSON` sees a `null` body it leaves `rawTx` as `nil`, and the very next line `rawTx.Decode` dereferences it, causing a panic.

### Impact Details

A null POST crashes the HTTP worker process for downstream clients such as VeWorld bridges, exchange or custodian relayers and other automation services.

## Proof of Concept

Use the test additions below in `thor/api/transactions/transactions_test.go`.

Add to the test map:

```go
"sendNullBodyCausesInternalError": sendNullBodyCausesInternalError,
```

Add the test function:

```go
func sendNullBodyCausesInternalError(t *testing.T) {
	res, statusCode, err := tclient.RawHTTPClient().RawHTTPPost("/transactions", []byte("null"))
	require.NoErrorf(t, err, "expected POST /transactions with null body to be handled gracefully, but request failed")
	assert.Equal(t, 400, statusCode, "null body should be rejected as bad request")
	assert.Contains(t, string(res), "body", "response should indicate invalid body")
}
```

Observed panic log:

```
panic serving 127.0.0.1:41950: runtime error: invalid memory address or nil pointer dereference
goroutine 36 [running]:
net/http.(*conn).serve.func1()
        /usr/local/go/src/net/http/server.go:1947 +0xbe
panic({0xb58940?, 0x12363d0?})
        /usr/local/go/src/runtime/panic.go:792 +0x132
github.com/vechain/thor/v2/api.(*RawTx).Decode(0xc0003f6280?)
```

{% stepper %}
{% step %}

### Steps to reproduce (test integration)

1. Add the test map entry:
   * `"sendNullBodyCausesInternalError": sendNullBodyCausesInternalError,`
2. Add the test function shown above to `thor/api/transactions/transactions_test.go`.
3. Run the test suite or perform a raw HTTP POST to `/transactions` with the body `null`.
4. Observe a panic / nil pointer dereference when `rawTx.Decode()` is called.
   {% endstep %}
   {% endstepper %}

## Notes

* The fixed behavior should validate that `rawTx` is non-nil before calling `Decode()` and return a proper 400 Bad Request indicating an invalid body.
* All links and references are preserved as in the original report.
