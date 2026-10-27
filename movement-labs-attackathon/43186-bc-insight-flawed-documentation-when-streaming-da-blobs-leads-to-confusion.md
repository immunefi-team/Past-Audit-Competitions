# #43186 \[BC-Insight] Flawed documentation when streaming da blobs leads to confusion

**Submitted on Apr 3rd 2025 at 13:11:23 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43186
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/da>
* **Impacts:**

## Description

### Description

Inside `stream_da_blobs_from_height`, the `certificate_stream` is obtained with the following call:

```javascript
let certificate_stream = self.stream_certificates().await?;
```

The stream is then iterated to retrieve individual `certificate` values:

```javascript
while let Some(certificate) = certificate_stream.next().await {
```

If a retrieved `certificate` returns `Ok` and `height > last_height`, the function proceeds to invoke `stream_da_blobs_between_heights`:

```javascript
Ok(Certificate::Height(height)) if height > last_height => {
    let blob_stream = self
        .stream_da_blobs_between_heights(last_height, height)
        .await?;
    tokio::pin!(blob_stream);

    while let Some(blob) = blob_stream.next().await {
        yield blob?;
    }

    last_height = height;
}
```

However, if `height <= last_height`, a warning message is logged instead:

```javascript
_ => {
    warn!("ignoring certificate");
}
```

It is important to note that the comment preceding this warning states:

```javascript
// If height is less than last height, ignore
```

This comment is incorrect because the warning is triggered not only when `height < last_height` but also when `height == last_height`.

### Impact

this fits the in-scope `Documentation Improvements` as the comment made here is incorrect.

### Recommendation

```diff
- // If height is less than last height, ignore
+ // If height is less than or equal to last height, ignore
```

## Proof of Concept

### POC

* A user or developer examining the codebase encounters the following `if` statement:

  ```javascript
  Ok(Certificate::Height(height)) if height > last_height => {
      let blob_stream = self
          .stream_da_blobs_between_heights(last_height, height)
          .await?;
      tokio::pin!(blob_stream);

      while let Some(blob) = blob_stream.next().await {
          yield blob?;
      }

      last_height = height;
  }
  ```
* Based on this logic, they reasonably assume that when the condition is not met, it means `height` is `<= last_height`.
* However, the documentation contains an incorrect comment that misleadingly states:

  ```javascript
  // If height is less than last height, ignore
  ```

  This is inaccurate because the warning is also triggered when `height == last_height`, not just when `height < last_height`.
