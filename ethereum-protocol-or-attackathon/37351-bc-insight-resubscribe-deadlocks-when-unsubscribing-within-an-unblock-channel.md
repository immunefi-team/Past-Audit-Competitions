# #37351 \[BC-Insight] Resubscribe Deadlocks When Unsubscribing Within An Unblock Channel

**Submitted on Dec 2nd 2024 at 17:36:30 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37351
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

`ResubscribeErr` located in Ethereum client Erigon ( <https://github.com/erigontech/erigon> ) is intended to perform event subscription. It waits for the subscription to fail and calls fn again. This process repeats until `Unsubscribe` is called or the active subscription ends successfully. The resub goroutine needs to live long enough to read from the unsub channel. Otherwise, an Unsubscribe call deadlocks when writing to the unblock unsub channel.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

In the event package, the `ResubscribeErr()` function is utilized to track the active subscription until `Unsubscribe` is called or the active subscription ends successfully:

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/event/subscription.go#L117>

```
func ResubscribeErr(backoffMax time.Duration, fn ResubscribeErrFunc) Subscription {
  s := &resubscribeSub{
    waitTime:   backoffMax / 10,
    backoffMax: backoffMax,
    fn:         fn,
    err:        make(chan error),
    unsub:      make(chan struct{}),
  }
  go s.loop()
  return s
}
```

As mentioned in the geth (go-ethereum) PR: <https://github.com/ethereum/go-ethereum/pull/28359>

Please note the unblock channel is used for unsubscribing, which is initialized with an inner goroutine call.

## Impact Details

If not properly structured in the resubscribe scenario, an Unsubscribe call can deadlock when attempting to write to the unblock unsub channel.

## References

* <https://github.com/erigontech/erigon/blob/v2.61.0-beta1>
* <https://github.com/ethereum/go-ethereum/pull/28359>

## Proof of Concept

## Proof of Concept

For simplicity, we can reuse the test from geth (go-ethereum) (<https://github.com/ethereum/go-ethereum/pull/28359>) to verify the issue:

```
package event


import (
   "context"
   "errors"
   "fmt"
   "reflect"
   "testing"
   "time"
)


func TestResubscribeWithCompletedSubscription(t *testing.T) {
   t.Parallel()


   quitProducerAck := make(chan struct{})
   quitProducer := make(chan struct{})


   sub := ResubscribeErr(100*time.Millisecond, func(ctx context.Context, lastErr error) (Subscription, error) {
      return NewSubscription(func(unsubscribed <-chan struct{}) error {
         select {
         case <-quitProducer:
            quitProducerAck <- struct{}{}
            return nil
         case <-unsubscribed:
            return nil
         }
      }), nil
   })


   // Ensure producer has started and exited before Unsubscribe
   close(quitProducer)
   <-quitProducerAck
   sub.Unsubscribe()
}
```

The test result shows a fatal error of `all goroutines are asleep - deadlock!` as follows:

```
=== RUN   TestResubscribeWithCompletedSubscription
=== PAUSE TestResubscribeWithCompletedSubscription
=== CONT  TestResubscribeWithCompletedSubscription
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
testing.tRunner.func1()
	/usr/local/go/src/testing/testing.go:1650 +0x4ab
testing.tRunner(0xc00010a4e0, 0xc000131c70)
	/usr/local/go/src/testing/testing.go:1695 +0x134
testing.runTests(0xc000010030, {0xabafc60, 0xc, 0xc}, {0x1?, 0xaa0e92e?, 0x0?})
	/usr/local/go/src/testing/testing.go:2159 +0x445
testing.(*M).Run(0xc0000780a0)
	/usr/local/go/src/testing/testing.go:2027 +0x68b
main.main()
	_testmain.go:79 +0x16c

goroutine 6 [chan send]:
github.com/ledgerwatch/erigon/event.(*resubscribeSub).Unsubscribe.func1()
	/Users/xxx/erigon/event/subscription.go:146 +0x25
sync.(*Once).doSlow(0xa95e952?, 0xc000024300?)
	/usr/local/go/src/sync/once.go:74 +0xc2
sync.(*Once).Do(...)
	/usr/local/go/src/sync/once.go:65
github.com/ledgerwatch/erigon/event.(*resubscribeSub).Unsubscribe(0xc000024360?)
	/Users/xxx/erigon/event/subscription.go:145 +0x3c
github.com/ledgerwatch/erigon/event.TestResubscribeWithCompletedSubscription(0xc00010a680?)
	/Users/xxx/erigon/event/subscription_test.go:49 +0xc3
testing.tRunner(0xc00010a680, 0xaada138)
	/usr/local/go/src/testing/testing.go:1689 +0xfb
created by testing.(*T).Run in goroutine 1
	/usr/local/go/src/testing/testing.go:1742 +0x390


Process finished with the exit code 1
```
