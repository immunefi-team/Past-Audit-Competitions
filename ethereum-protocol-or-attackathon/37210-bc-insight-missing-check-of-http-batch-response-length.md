# #37210 \[BC-Insight] Missing Check of HTTP Batch Response Length

**Submitted on Nov 28th 2024 at 20:53:10 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37210
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The Client.BatchCallContext in the rpc package of Ethereum client Erigon ( <https://github.com/erigontech/erigon> ) could be stuck due to the missing check of the HTTP batch response length in the sendBatchHTTP().

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

In the rpc package, the Client.BatchCall() is used to send all requests in a single batch, which calls the Client.BatchCallContext() to perform the batched calls, in which the Client.sendBatchHTTP() will be invoked in case that it’s an HTTP request:

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/rpc/client.go#L332>

```
// BatchCall sends all given requests as a single batch and waits for the server
// to return a response for all of them.
//
// In contrast to Call, BatchCall only returns I/O errors. Any error specific to
// a request is reported through the Error field of the corresponding BatchElem.
//
// Note that batch calls may not be executed atomically on the server side.
func (c *Client) BatchCall(b []BatchElem) error {
   ctx := context.Background()
   return c.BatchCallContext(ctx, b)
}


// BatchCall sends all given requests as a single batch and waits for the server
// to return a response for all of them. The wait duration is bounded by the
// context's deadline.
//
// In contrast to CallContext, BatchCallContext only returns errors that have occurred
// while sending the request. Any error specific to a request is reported through the
// Error field of the corresponding BatchElem.
//
// Note that batch calls may not be executed atomically on the server side.
func (c *Client) BatchCallContext(ctx context.Context, b []BatchElem) error {
   msgs := make([]*jsonrpcMessage, len(b))
   op := &requestOp{
      ids:  make([]json.RawMessage, len(b)),
      resp: make(chan *jsonrpcMessage, len(b)),
   }
   for i, elem := range b {
      msg, err := c.newMessage(elem.Method, elem.Args...)
      if err != nil {
         return err
      }
      msgs[i] = msg
      op.ids[i] = msg.ID
   }


   var err error
   if c.isHTTP {
      err = c.sendBatchHTTP(ctx, op, msgs)
   } else {
      err = c.send(ctx, op, msgs)
   }


   // Wait for all responses to come back.
   for n := 0; n < len(b) && err == nil; n++ {
      var resp *jsonrpcMessage
      resp, err = op.wait(ctx, c)
      if err != nil {
         break
      }
      // Find the element corresponding to this response.
      // The element is guaranteed to be present because dispatch
      // only sends valid IDs to our channel.
      var elem *BatchElem
      for i := range msgs {
         if bytes.Equal(msgs[i].ID, resp.ID) {
            elem = &b[i]
            break
         }
      }
      if resp.Error != nil {
         elem.Error = resp.Error
         continue
      }
      if len(resp.Result) == 0 {
         elem.Error = ErrNoResult
         continue
      }
      elem.Error = json.Unmarshal(resp.Result, elem.Result)
   }
   return err
}
```

As mentioned in the PR: <https://github.com/ethereum/go-ethereum/pull/26064>

> It turns out that Client.BatchCallContext relies on the number or response messages exactly matching the number of requests. If too many responses are received, sendBatchHTTP blocks trying to send more than will fit in the channel buffer, and never yields to timeout. If too few responses are received, BatchCallContext waits for the missing responses from the empty channel, but eventually yields to timeout.

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/rpc/http.go#L125>

```
func (c *Client) sendBatchHTTP(ctx context.Context, op *requestOp, msgs []*jsonrpcMessage) error {
   hc := c.writeConn.(*httpConn)
   respBody, err := hc.doRequest(ctx, msgs)
   if err != nil {
      return err
   }
   var respmsgs []jsonrpcMessage
   if err := json.Unmarshal(respBody, &respmsgs); err != nil {
      return err
   }


   for i := 0; i < len(respmsgs); i++ {
      op.resp <- &respmsgs[i]
   }
   return nil
}
```

It is worth noted a similar issue has been fixed in go-ethereum by checking the message and response have same size : <https://github.com/ethereum/go-ethereum/pull/26064>

```
func (c *Client) sendBatchHTTP(ctx context.Context, op *requestOp, msgs []*jsonrpcMessage) error {
   hc := c.writeConn.(*httpConn)
   respBody, err := hc.doRequest(ctx, msgs)
   if err != nil {
      return err
   }
   var respmsgs []jsonrpcMessage
   if err := json.Unmarshal(respBody, &respmsgs); err != nil {
      return err
   }


   if len(respmsgs) != len(msgs) {
       return fmt.Errorf("batch has %d requests but response has %d: %w", len(msgs), len(respmsgs), ErrBadResult)
   }


   for i := 0; i < len(respmsgs); i++ {
      op.resp <- &respmsgs[i]
   }
   return nil
}
```

## Impact Details

Client.BatchCall() could be stuck in case two many or too few responses are received.

## References

* <https://github.com/erigontech/erigon/blob/v2.61.0-beta1>
* <https://github.com/ethereum/go-ethereum/pull/26064>

## Proof of Concept

## Proof of Concept

For simplicity, we can reuse the test from go-ethereum (<https://github.com/ethereum/go-ethereum/pull/26064> ) to verify the issue:

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


func TestClientBatchRequest_len(t *testing.T) {
   logger := log.New()
   b, err := json.Marshal([]jsonrpcMessage{
      {Version: "2.0", ID: json.RawMessage("1"), Method: "foo", Result: json.RawMessage(`"0x1"`)},
      {Version: "2.0", ID: json.RawMessage("2"), Method: "bar", Result: json.RawMessage(`"0x2"`)},
   })
   if err != nil {
      t.Fatal("failed to encode jsonrpc message:", err)
   }
   s := httptest.NewServer(http.HandlerFunc(func(rw http.ResponseWriter, r *http.Request) {
      _, err := rw.Write(b)
      if err != nil {
         t.Error("failed to write response:", err)
      }
   }))
   t.Cleanup(s.Close)


   client, err := Dial(s.URL, logger)
   if err != nil {
      t.Fatal("failed to dial test server:", err)
   }
   defer client.Close()


   t.Run("too-few", func(t *testing.T) {
      batch := []BatchElem{
         {Method: "foo"},
         {Method: "bar"},
         {Method: "baz"},
      }
      ctx, cancelFn := context.WithTimeout(context.Background(), time.Second)
      defer cancelFn()
      if err := client.BatchCallContext(ctx, batch); !errors.Is(err, ErrBadResult) {
         t.Errorf("expected %q but got: %v", ErrBadResult, err)
      }
   })


   t.Run("too-many", func(t *testing.T) {
      batch := []BatchElem{
         {Method: "foo"},
      }
      ctx, cancelFn := context.WithTimeout(context.Background(), time.Second)
      defer cancelFn()
      if err := client.BatchCallContext(ctx, batch); !errors.Is(err, ErrBadResult) {
         t.Errorf("expected %q but got: %v", ErrBadResult, err)
      }
   })
}
```

The test hangs and does not complete.
