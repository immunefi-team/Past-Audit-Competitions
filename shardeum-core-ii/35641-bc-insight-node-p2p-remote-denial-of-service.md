# #35641 \[BC-Insight] node p2p remote denial of service

**Submitted on Oct 1st 2024 at 21:39:01 UTC by @gln for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35641
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://immunefi.com>
* **Impacts:**
  * Causing network processing nodes to process transactions from the transaction queue beyond set parameters
  * Direct loss of funds

## Description

## Brief/Intro

Shardeum node will read unsigned integer from a network and allocated memory without any bounds checks.

## Vulnerability Details

There are a lot of places in lib-net, where uint32 is read from an incoming packet and buffer of requested size is allocated.

Consider this example <https://github.com/shardeum/lib-net/blob/dev/shardus\\_net/src/message.rs#L96>

\`\`\` pub fn deserialize(cursor: \&mut Cursor\<Vec\<u8>>) -> Option\<Message> { let mut header\_version\_bytes = \[0u8; 1]; cursor.read\_exact(\&mut header\_version\_bytes).ok()?; let header\_version = u8::from\_le\_bytes(header\_version\_bytes);

```
    let mut header_len_bytes &#x3D; [0u8; 4];
    cursor.read_exact(&amp;mut header_len_bytes).ok()?;
```

1. ```
      let header_len &#x3D; u32::from_le_bytes(header_len_bytes);
   ```
2. ```
      let mut header_bytes &#x3D; vec![0u8; header_len as usize];
    cursor.read_exact(&amp;mut header_bytes).ok()?;
    ...
   ```

\`\`\`

1. 'header\_len' does not have any upper bounds checks.
2. so 4GB buffer will be allocated

If several such requests will be very close in time, memory allocation will fail and rust panics.

As a result node will crash with out of memory error.

## Impact Details

An attacker will be able to trigger out of memory error and crash shardeum node remotely.

## Proof of Concept

## Proof of Concept

How to reproduce:

1. get test1.js by using provided gist link <https://gist.github.com/gln7/ad3815f298bb898d3161bba8cb8999bc>

(basically, this is the same as test.js from report #34093, thanks to author!)

2. get and build lib-net:

\`\`\` $ git clone <https://github.com/shardeum/lib-net> --depth 1 $ cd lib-net $ npm run build

\`\`\`

3. run test server: \`\`\` $ ulimit -Sv 4000000 $ node test1.js \`\`\`
4. get t1.py (gist link - <https://gist.github.com/gln7/99e0f927db93e70c690c5893007e63e3>)
5. run t1.py:

\`\`\` ./t1.py localhost 10001 \`\`\`

6. nodejs will crash:

\`\`\` memory allocation of 4294967295 bytes failed Aborted (core dumped)

\`\`\`
