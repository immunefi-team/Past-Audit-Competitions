# #38920 \[BC-Medium] teku remote DoS

**Submitted on Jan 18th 2025 at 00:07:35 UTC by @gln for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38920
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/Consensys/teku>
* **Impacts:**
  * Increasing greater than or equal to 25% of network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

Teku does not validate parameters of incoming BlobSidecarsByRange requests.

Malformed BlobSidecarsByRange request allows an attacker to bypass validateRequest() method and also approveObjectsCount() (internal rate limiter).

## Vulnerability Details

Let's look at the code which process req/resp requests <https://github.com/Consensys/teku/blob/master/networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/rpc/core/Eth2IncomingRequestHandler.java#L100>

```
private void handleRequest(
      final Optional<Eth2Peer> peer,
      final TRequest request,
      final ResponseCallback<TResponse> callback) {
    try {
      requestHandled.set(true);
      final Optional<RpcException> requestValidationError =
1)          localMessageHandler.validateRequest(protocolId, request);
      if (requestValidationError.isPresent()) {
        callback.completeWithErrorResponse(requestValidationError.get());
        return;
      }
2)      localMessageHandler.onIncomingMessage(protocolId, peer, request, callback);
    } catch (final StreamClosedException e) {
	...
```

1. Corresponding validateRequest() method is called to verify the request
2. If validateRequest() do not return error, actual message handler is called

Let's look at BlobSidecarsByRange handler <https://github.com/Consensys/teku/blob/master/networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/rpc/beaconchain/methods/BlobSidecarsByRangeMessageHandler.java#L85>

```
public Optional<RpcException> validateRequest(
      final String protocolId, final BlobSidecarsByRangeRequestMessage request) {

    final SpecConfigDeneb specConfig =
        SpecConfigDeneb.required(spec.atSlot(request.getMaxSlot()).getConfig());

    final int maxRequestBlobSidecars = specConfig.getMaxRequestBlobSidecars();
    final int maxBlobsPerBlock = specConfig.getMaxBlobsPerBlock();

1)    final long requestedCount = calculateRequestedCount(request, maxBlobsPerBlock);

3)    if (requestedCount > maxRequestBlobSidecars) {
      requestCounter.labels("count_too_big").inc();
      return Optional.of(
          new RpcException(
              INVALID_REQUEST_CODE,
              String.format(
                  "Only a maximum of %s blob sidecars can be requested per request",
                  maxRequestBlobSidecars)));
    }

    return Optional.empty();
}

private long calculateRequestedCount(
      final BlobSidecarsByRangeRequestMessage message, final int maxBlobsPerBlock) {
2)    return maxBlobsPerBlock * message.getCount().longValue();
}


```

1. To verify request parameters calculateRequestedCount is called
2. Basically message.getCount() returns uint64, the type of return value is long, if message.getCount() is large enough, integer overflow will occur and result will be negative
3. This check will pass if requestedCount is negative

Thus validateRequest() checks can be bypassed.

Let's look at BlobSidecarsByRange handler, there are few more checks <https://github.com/Consensys/teku/blob/master/networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/rpc/beaconchain/methods/BlobSidecarsByRangeMessageHandler.java#L110>

```
 public void onIncomingMessage(
      final String protocolId,
      final Eth2Peer peer,
      final BlobSidecarsByRangeRequestMessage message,
      final ResponseCallback<BlobSidecar> callback) {
    final UInt64 startSlot = message.getStartSlot();
    final UInt64 endSlot = message.getMaxSlot();

    final SpecConfigDeneb specConfig = SpecConfigDeneb.required(spec.atSlot(endSlot).getConfig());
1)    final long requestedCount = calculateRequestedCount(message, specConfig.getMaxBlobsPerBlock());

    final Optional<RequestApproval> blobSidecarsRequestApproval =
2)        peer.approveBlobSidecarsRequest(callback, requestedCount);

3)    if (!peer.approveRequest() || blobSidecarsRequestApproval.isEmpty()) {
      requestCounter.labels("rate_limited").inc();
      return;
    }

    requestCounter.labels("ok").inc();
    totalBlobSidecarsRequestedCounter.inc(message.getCount().longValue());
	...
```

1. Assuming message is malformed, so calculateRequestedCount() will return negative value
2. We need to look at approveBlobSidecarsRequest() method <https://github.com/Consensys/teku/blob/master/networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/peers/DefaultEth2Peer.java#L386>

```
 public Optional<RequestApproval> approveBlobSidecarsRequest(
      final ResponseCallback<BlobSidecar> callback, final long blobSidecarsCount) {
    return approveObjectsRequest(
        "blob sidecars", blobSidecarsRequestTracker, blobSidecarsCount, callback);
  }
```

The approveObjectsRequest() method will finally call this method <https://github.com/Consensys/teku/blob/master/networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/peers/RateTrackerImpl.java#L48>

```
public synchronized Optional<RequestApproval> approveObjectsRequest(final long objectsCount) {
    pruneRequests();
    final UInt64 currentTime = timeProvider.getTimeInSeconds();
1)    if (peerRateLimit - objectsWithinWindow <= 0) {
      return Optional.empty();
    }
2)    objectsWithinWindow += objectsCount;
    final RequestApproval requestApproval =
        new RequestApproval.RequestApprovalBuilder()
            .requestId(newRequestId++)
            .timeSeconds(currentTime)
            .objectsCount(objectsCount)
            .build();
    requests.put(requestApproval.getRequestKey(), objectsCount);
    return Optional.of(requestApproval);
  }
```

1. if objectsWithinWindow is negative, this check will not pass
2. here objectsWithinWindow can be made negative (if objectCount is negative)

As a result, negative objectsCount allows us to bypass internal rate limiter and request as many blobs as there are available.

## Impact Details

Maximum amount of space occupied by blobs is around 50GB <https://lighthouse-book.sigmaprime.io/advanced-blobs.html>

This is a potential Denial of Service issue, as an attacker could request huge amount of blobs from a teku node.

## Link to Proof of Concept

<https://gist.github.com/gln7/1b92502a6a24b8af4dab151c7f0b6827>

## Proof of Concept

## Proof of Concept

To reproduce issue we need to patch corresponding test and verify that validateRequest() checks can by bypassed.

How to reproduce:

1. get teku source

```
$ git rev-parse master
d56ce97f4de3f85e739a7499bad29871c79b2c03

```

2. apply poc.patch
3. run test:

```
$ ./gradlew -i test --tests "BlobSidecarsByRangeMessageHandlerTest.shouldNotSendBlobSidecarsIfPeerIsRateLimited*" > 1.log 2>&1

$ grep XXXKE 1.log

```

You should see output like this:

```
    XXXXKE requesting blobs for slots 1..1000
    XXXXKE validation_result=Optional[tech.pegasys.teku.networking.eth2.rpc.core.RpcException: [Code 1] Only a maximum of 768 blob sidecars can be requested per request]

    XXXKE requesting blobs for slots 1..(MAX_VALUE-100000)
    XXXXKE validateRequest OK
    XXXXKE validation_result=Optional.empty
    XXXXKE sendBlobSidecars call...

```

1. when trying to request 1000 blob sidecars, validateRequest() will return error
2. when trying to request uint64.MAX\_VALUE-1000000 sidecars, valideRequest() check will pass and sendBlobSidecars() will be called
