# #37350 \[BC-Insight] \`null\` Is Not Unmarshalled Correctly Into json.RawMessage

**Submitted on Dec 2nd 2024 at 17:25:43 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37350
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The response of `null` in the rpc package of Ethereum client Erigon ( <https://github.com/erigontech/erigon> ) could not be presented correctly due to the unmarshalling of the `null` in the CallContext.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

In the rpc package, the CallContext is used to perform a JSON-RPC call and unmarshal the returned result. However, the unmarshalling for the result of `null` is not correctly performed:

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/rpc/client.go#L321>

```
// CallContext performs a JSON-RPC call with the given arguments. If the context is
// canceled before the call has successfully returned, CallContext returns immediately.
//
// The result must be a pointer so that package json can unmarshal into it. You
// can also pass nil, in which case the result is ignored.
func (c *Client) CallContext(ctx context.Context, result interface{}, method string, args ...interface{}) error {
  if result != nil && reflect.TypeOf(result).Kind() != reflect.Ptr {
    return fmt.Errorf("call result parameter must be pointer or nil interface: %v", result)
  }
  msg, err := c.newMessage(method, args...)
  if err != nil {
    return err
  }
  op := &requestOp{ids: []json.RawMessage{msg.ID}, resp: make(chan *jsonrpcMessage, 1)}


  if c.isHTTP {
    err = c.sendHTTP(ctx, op, msg)
  } else {
    err = c.send(ctx, op, msg)
  }
  if err != nil {
    return err
  }


  // dispatch has accepted the request and will close the channel when it quits.
  switch resp, err := op.wait(ctx, c); {
  case err != nil:
    return err
  case resp.Error != nil:
    return resp.Error
  case len(resp.Result) == 0:
    return ErrNoResult
  default:
    return json.Unmarshal(resp.Result, &result)
  }
}
```

As mentioned in the geth(go-ethereum) PR: <https://github.com/ethereum/go-ethereum/pull/26701>

> The function already checks that the result is either `nil` or a pointer type, so the extra reference operator is unnecessary. This actually causes a bug where `null`s are not unmarshalled correctly into json.RawMessage.

It is worth noted a similar issue has been fixed in geth (go-ethereum) by fixing the result unmarshalling dependent of `null`:

<https://github.com/ethereum/go-ethereum/pull/26723>

```
func (c *Client) CallContext(ctx context.Context, result interface{}, method string, args ...interface{}) error {
   ...
  default:
    return json.Unmarshal(resp.Result, &result)
    if result == nil {
      return nil
    }
    return json.Unmarshal(resp.Result, result)
  }
```

## Impact Details

`null` result could not be unmarshalled correctly in the json.RawMessage in the CallContext of rpc.

## References

* <https://github.com/erigontech/erigon/blob/v2.61.0-beta1>
* <https://github.com/ethereum/go-ethereum/issues/26700>
* <https://github.com/ethereum/go-ethereum/pull/26701>
* <https://github.com/ethereum/go-ethereum/pull/26723>

## Proof of Concept

## Proof of Concept

For simplicity, we can reuse and modify the test from geth (go-ethereum) (<https://github.com/ethereum/go-ethereum/pull/26723>) to verify the issue:

```
package rpc


import (
   "context"
   "encoding/json"
   "errors"
   "fmt"
   "math/rand"
   "net"
   "net/http"
   "net/http/httptest"
   "reflect"
   "strings"
   "sync"
   "testing"
   "time"


   "github.com/davecgh/go-spew/spew"
   "github.com/ledgerwatch/erigon-lib/common/dbg"
   "github.com/ledgerwatch/log/v3"
)




type nullTestService struct{}


func TestNullResponse(t *testing.T) {
   logger := log.New()
   server := newTestServer(logger)
   defer server.Stop()
   err := server.RegisterName("test", new(nullTestService))


   if err != nil {


      t.Fatal(err)
   }


   client := DialInProc(server, logger)
   defer client.Close()
   result := &jsonrpcMessage{}


   err = client.Call(&result.Result, "test_returnNull")
   if err != nil {
      t.Fatal(err)
   }


   if result.Result == nil {
      t.Fatal("Expected non-nil result")
   }


   if !reflect.DeepEqual(result.Result, json.RawMessage("null")) {
      t.Errorf("Expected null, got %s", result.Result)
   }
}
```

The output shows the `Expected non-nil result` error message when the result is nil.

```
=== RUN   TestNullResponse
[WARN] [12-02|12:17:37.643] Cannot register RPC callback [invalidRets1] - error must the last return value 
[WARN] [12-02|12:17:37.643] Cannot register RPC callback [invalidRets2] - error must the last return value 
[WARN] [12-02|12:17:37.643] Cannot register RPC callback [invalidRets3] - maximum 2 return values are allowed, got 3 
    client_test.go:140: Expected non-nil result
--- FAIL: TestNullResponse (0.00s)

FAIL
```
