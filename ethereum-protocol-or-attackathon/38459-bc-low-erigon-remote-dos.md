# #38459 \[BC-Low] erigon remote DoS

**Submitted on Jan 3rd 2025 at 22:39:50 UTC by @gln for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38459
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

Erigon contains embedded consensus implementation called Caplin.

BlobSidecarsByRange handler does not validate incoming parameters.

Malformed p2p request allows an attacker to trigger infinite loop in erigon node.

## Vulnerability Details

Let's look at the BlobSidecarsByRange p2p handler <https://github.com/erigontech/erigon/blob/main/cl/sentinel/handlers/blobs.go#L32>

```
func (c *ConsensusHandlers) blobsSidecarsByRangeHandler(s network.Stream) error {

	req := &cltypes.BlobsByRangeRequest{}
	if err := ssz_snappy.DecodeAndReadNoForkDigest(s, req, clparams.DenebVersion); err != nil {
		return err
	}

	tx, err := c.indiciesDB.BeginRo(c.ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	written := 0
1)	for slot := req.StartSlot; slot < req.StartSlot+req.Count; slot++ {
2)		blockRoot, err := beacon_indicies.ReadCanonicalBlockRoot(tx, slot)
		if err != nil {
			return err
		}
3)		if blockRoot == (libcommon.Hash{}) {
			continue
		}

		blobCount, err := c.blobsStorage.KzgCommitmentsCount(c.ctx, blockRoot)
		if err != nil {
			return err
		}

		for i := 0; i < int(blobCount) && written < maxBlobsThroughoutputPerRequest; i++ {
	    ...
        }
    }
	return nil
}
```

1. req.StartSlot and req.Count are not validated
2. ReadCanonicalBlobkRoot() returns zero hash for invalid slot
3. the 'for' loop continues to run

Basially, if req.StarSlot and req.Count are large enough, erigon could enter into an infinite loop.

## Impact Details

Attacker could trigger an infinite loop in erigon node with a single malformed p2p request.

## Link to Proof of Concept

<https://gist.github.com/gln7/c8106d435f0d1f3cde96f8d76e886e10>

## Proof of Concept

## Proof of Concept

How to reproduce:

1. get erigon source

```
$ git rev-parse main
8945a32131b8d6d0c0a344d43ed6cc04b0f5944c

```

2. apply poc.patch (see gist link)
3. run test:

```
$ cd cl/sentinel/handlers
$ go test -run TestBlobsByRangeHandler 

```

Golang enters into an infite loop and starts to output messages like these:

```
XXXKE blobsSidecarsByRangeHandler slot 1095042206
XXXKE blobsSidecarsByRangeHandler slot 1095042207
XXXKE blobsSidecarsByRangeHandler slot 1095042208
XXXKE blobsSidecarsByRangeHandler slot 1095042209
XXXKE blobsSidecarsByRangeHandler slot 1095042210
XXXKE blobsSidecarsByRangeHandler slot 1095042211
XXXKE blobsSidecarsByRangeHandler slot 1095042212
...
```
