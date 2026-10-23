# #37246 \[BC-Low] lodestar snappy checksum issue

**Submitted on Nov 29th 2024 at 22:23:56 UTC by @gln for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37246
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/chainsafe/lodestar>
* **Impacts:**
  * Unintended permanent chain split affecting greater than or equal to 25% of the network, requiring hard fork (network partition requiring hard fork)

## Description

## Brief/Intro

Lodestar does not verify checksum in snappy framing uncompressed chunks.

## Vulnerability Details

In Req/Resp protocol the messages are encoded by using ssz\_snappy encoding,\
which is a snappy framing compression over ssz encoded message.

In snappy framing format there are uncompressed chunks, each such chunk is prefixed with a checksum.

Let's see how golang implementation parses such chunks - <https://github.com/golang/snappy/blob/master/decode.go#L176>

```
	case chunkTypeUncompressedData:
			// Section 4.3. Uncompressed data (chunk type 0x01).
			if chunkLen < checksumSize {
				r.err = ErrCorrupt
				return r.err
			}
			buf := r.buf[:checksumSize]
			if !r.readFull(buf, false) {
				return r.err
			}
			checksum := uint32(buf[0]) | uint32(buf[1])<<8 | uint32(buf[2])<<16 | uint32(buf[3])<<24
			// Read directly into r.decoded instead of via r.buf.
			n := chunkLen - checksumSize
			if n > len(r.decoded) {
				r.err = ErrCorrupt
				return r.err
			}
			if !r.readFull(r.decoded[:n], false) {
				return r.err
			}
			if crc(r.decoded[:n]) != checksum {
				r.err = ErrCorrupt
				return r.err
			}
			r.i, r.j = 0, n
			continue
```

As you can see, if checksum is incorrect, decoder fails and returns error.

Now let's look at lodestar decoder <https://github.com/ChainSafe/lodestar/blob/unstable/packages/reqresp/src/encodingStrategies/sszSnappy/snappyFrames/uncompress.ts#L17>

```
uncompress(chunk: Uint8ArrayList): Uint8ArrayList | null {
    this.buffer.append(chunk);
    const result = new Uint8ArrayList();
    while (this.buffer.length > 0) {
      if (this.buffer.length < 4) break;

      const type = getChunkType(this.buffer.get(0));
      const frameSize = getFrameSize(this.buffer, 1);

      if (this.buffer.length - 4 < frameSize) {
        break;
      }

      const data = this.buffer.subarray(4, 4 + frameSize);
      this.buffer.consume(4 + frameSize);

      if (!this.state.foundIdentifier && type !== ChunkType.IDENTIFIER) {
        throw "malformed input: must begin with an identifier";
      }

      if (type === ChunkType.IDENTIFIER) {
        if (!Buffer.prototype.equals.call(data, IDENTIFIER)) {
          throw "malformed input: bad identifier";
        }
        this.state.foundIdentifier = true;
        continue;
      }

      if (type === ChunkType.COMPRESSED) {
        result.append(uncompress(data.subarray(4)));
      }
      if (type === ChunkType.UNCOMPRESSED) {
1)        result.append(data.subarray(4));
      }
    }
    if (result.length === 0) {
      return null;
    }
    return result;
  }

```

1. As you can see, checksum is not verified, bytes are appended to 'result'

## Impact Details

Faulty nodes may trigger chain stall by sending p2p messages with incorrect checksum. Such messages will be parsed by lodestar clients by will be rejected by another implementations.

## Link to Proof of Concept

<https://gist.github.com/gln7/aab55674431b1c8d42a59ccf9d7cbf60>

## Proof of Concept

## Proof of Concept

How to reproduce:

1. get poc via gist link and run it:

```
$ node dec1.mjs 
checking chunk type=255
checking chunk type=1
got uncompressed chunk..
Decompressed ok 124 bytes

```
