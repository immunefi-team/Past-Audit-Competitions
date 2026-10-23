# Boost \_ Firedancer v0.1 34234 - \[Blockchain\_DLT - Insight] Setting the variable shred\_cnt in the shr

Submitted on Wed Aug 07 2024 07:06:41 GMT-0400 (Atlantic Standard Time) by @c4a4dda89 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #34234

Report type: Blockchain/DLT

Report severity: Insight

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

In the subsequent `after_frag` code, When the value of the member variable shred\_cnt within the shred34 object is set to 0, all boundary checks are bypassed.

Attention must be drawn to the fact that the matter under discussion is not synonymous with the issue delineated within report `33718`.

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
  if( FD_LIKELY( shred34->shred_cnt ) ) { // shred34->shred_cnt == 0，all check bypass here
    FD_TEST( shred34->offset<*opt_sz  );
    FD_TEST( shred34->shred_cnt<=34UL );
    FD_TEST( shred34->stride==sizeof(shred34->pkts[0]) );
  }
 //...
 
}

```

## Impact Details

Process-to-process memory corruption may lead to the process-to-process RCE between sandboxed tiles.

## References

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_store.c#L101>

## Proof of concept

## Proof of Concept

The attack surface of this vulnerability is when an attacker has arbitrary code execution rights over `fd_shred`, and then launches a process to process RCE attack on `fd_store`. Therefore, we modify the relevant code of the `fd_shred` process to simulate the situation where the attacker has already obtained the ability to execute code.

The project side realized that the modified content shown by the git diff needs to be synchronized to the local environment. By executing `make -j fddev` and sudo `fddev --no-sandbox`, Owing to the arrangement of the memory layout on the heap, the proof of concept (PoC) might not precipitate a system crash; however, upon debugging, it becomes evident that a distorted instance of `shred34` is being erroneously conveyed to subsequent functions for further processing.

```shell
@@ -602,6 +615,11 @@ after_frag( void *             _ctx,
   /* Add whatever is left to the last shred34 */
   s34[ fd_ulong_if( s34[ 3 ].shred_cnt>0UL, 3, 2 ) ].est_txn_cnt += ctx->shredded_txn_cnt - txn_per_s34*s34_cnt;

+
+  s34[0].shred_cnt=0;
+  s34[0].offset=0xdeadbeefdeadbeef;
+  s34[0].stride=0x4141414141414141;
+
   /* Send to the blockstore, skipping any empty shred34_t s. */
   ulong sig = in_idx!=NET_IN_IDX; /* sig==0 means the store tile will do extra checks */
```
