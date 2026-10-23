# Boost \_ Firedancer v0.1 34564 - \[Blockchain\_DLT - Medium] shred tile overflow

Submitted on Thu Aug 15 2024 21:52:58 GMT-0400 (Atlantic Standard Time) by @gln for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #34564

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Any bug leading to loss of funds or acceptance of forged / invalid signatures

## Description

## Brief/Intro

To process incoming shreds from network, shred tile calls fd\_fec\_resolver\_add\_shred() which is vulnerable to heap overflow.

## Vulnerability Details

Let's look at the code <https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/run/tiles/fd\\_shred.c#L298>

```

static void
during_frag( void * _ctx,
             ulong  in_idx,
             ulong  seq,
             ulong  sig,
             ulong  chunk,
             ulong  sz,
             int *  opt_filter ) {
  (void)seq;

  fd_shred_ctx_t * ctx = (fd_shred_ctx_t *)_ctx;
  ...
   } else { /* the common case, from the netmux tile */
    /* The FEC resolver API does not present a prepare/commit model. If we
       get overrun between when the FEC resolver verifies the signature
       and when it stores the local copy, we could end up storing and
       retransmitting garbage.  Instead we copy it locally, sadly, and
       only give it to the FEC resolver when we know it won't be overrun
       anymore. */
1.    if( FD_UNLIKELY( chunk<ctx->net_in_chunk0 || chunk>ctx->net_in_wmark || sz>FD_NET_MTU ) )
      FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->net_in_chunk0, ctx->net_in_wmark ));
    uchar const * dcache_entry = fd_chunk_to_laddr_const( ctx->net_in_mem, chunk );
    ulong hdr_sz = fd_disco_netmux_sig_hdr_sz( sig );
    FD_TEST( hdr_sz <= sz ); /* Should be ensured by the net tile */
    fd_shred_t const * shred = fd_shred_parse( dcache_entry+hdr_sz, sz-hdr_sz );
    if( FD_UNLIKELY( !shred ) ) {
      *opt_filter = 1;
      return;
    };
    ...
    fd_memcpy( ctx->shred_buffer, dcache_entry+hdr_sz, sz-hdr_sz );
    ctx->shred_buffer_sz = sz-hdr_sz;
  }
}
```

1. The only check here is that packet size should be larger than FD\_NET\_MTU, which is 2048

Now let's look at function after\_frag(), which processes incoming shreds:

```
static void
after_frag( void *             _ctx,
            ulong              in_idx,
            ulong              seq,
            ulong *            opt_sig,
            ulong *            opt_chunk,
            ulong *            opt_sz,
            ulong *            opt_tsorig,
            int *              opt_filter,
            fd_mux_context_t * mux ) {

  fd_shred_ctx_t * ctx = (fd_shred_ctx_t *)_ctx;

  ...
  const ulong fanout = 200UL;
  fd_shred_dest_idx_t _dests[ 200*(FD_REEDSOL_DATA_SHREDS_MAX+FD_REEDSOL_PARITY_SHREDS_MAX) ];

  if( FD_LIKELY( in_idx==NET_IN_IDX ) ) {
    uchar * shred_buffer    = ctx->shred_buffer;
    ulong   shred_buffer_sz = ctx->shred_buffer_sz;

1.    fd_shred_t const * shred = fd_shred_parse( shred_buffer, shred_buffer_sz );
    if( FD_UNLIKELY( !shred       ) ) { ctx->metrics->shred_processing_result[ 1 ]++; return; }

    fd_epoch_leaders_t const * lsched = fd_stake_ci_get_lsched_for_slot( ctx->stake_ci, shred->slot );
    if( FD_UNLIKELY( !lsched      ) ) { ctx->metrics->shred_processing_result[ 0 ]++; return; }

    fd_pubkey_t const * slot_leader = fd_epoch_leaders_get( lsched, shred->slot );
    if( FD_UNLIKELY( !slot_leader ) ) { ctx->metrics->shred_processing_result[ 0 ]++; return; } /* Count this as bad slot too */

    fd_fec_set_t const * out_fec_set[ 1 ];
    fd_shred_t   const * out_shred[ 1 ];

    long add_shred_timing  = -fd_tickcount();
2.    int rv = fd_fec_resolver_add_shred( ctx->resolver, shred, shred_buffer_sz, slot_leader->uc, out_fec_set, out_shred );
    add_shred_timing      +=  fd_tickcount();

    fd_histf_sample( ctx->metrics->add_shred_timing, (ulong)add_shred_timing );
    ctx->metrics->shred_processing_result[ rv + FD_FEC_RESOLVER_ADD_SHRED_RETVAL_OFF+FD_SHRED_ADD_SHRED_EXTRA_RETVAL_CNT ]++;

    if( (rv==FD_FEC_RESOLVER_SHRED_OKAY) | (rv==FD_FEC_RESOLVER_SHRED_COMPLETES) ) {
      /* Relay this shred */
      ulong fanout = 200UL;
      ulong max_dest_cnt[1];
      do {
        /* If we've validated the shred and it COMPLETES but we can't
           compute the destination for whatever reason, don't forward
           the shred, but still send it to the blockstore. */
        fd_shred_dest_t * sdest = fd_stake_ci_get_sdest_for_slot( ctx->stake_ci, shred->slot );
        if( FD_UNLIKELY( !sdest ) ) break;
        fd_shred_dest_idx_t * dests = fd_shred_dest_compute_children( sdest, &shred, 1UL, _dests, 1UL, fanout, fanout, max_dest_cnt );
        if( FD_UNLIKELY( !dests ) ) break;

        for( ulong j=0UL; j<*max_dest_cnt; j++ ) send_shred( ctx, *out_shred, sdest, dests[ j ], ctx->tsorig );
      } while( 0 );
    }
    if( FD_LIKELY( rv!=FD_FEC_RESOLVER_SHRED_COMPLETES ) ) return;

    FD_TEST( ctx->fec_sets <= *out_fec_set );
    ctx->send_fec_set_idx = (ulong)(*out_fec_set - ctx->fec_sets);
    ctx->shredded_txn_cnt = 0UL;
  } else {
    /* We know we didn't get overrun, so advance the index */
    ctx->shredder_fec_set_idx = (ctx->shredder_fec_set_idx+1UL)%ctx->shredder_max_fec_set_idx;
  }

```

1. The shred is parsed by calling fd\_shred\_parse().

Note that fd\_shred\_parse() basically does not have any upper bounds limits on incoming shreds.

As a result, shred could have any size between FD\_SHRED\_MAX\_SZ (which is 1228 bytes) and FD\_NET\_MTU.

2. To add parsed shred to FEC set, the function fd\_fec\_resolver\_add\_shred() is called

Let's look at this function:

```
int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
                               fd_shred_t   const   * shred,
                               ulong                  shred_sz,
                               uchar        const   * leader_pubkey,
                               fd_fec_set_t const * * out_fec_set,
                               fd_shred_t   const * * out_shred ) {

  ...
  if( FD_UNLIKELY( ctx_map_key_inval( *w_sig ) ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;

  /* Are we already done with this FEC set? */
  int found = !!ctx_map_query( done_map, *w_sig, NULL );
  if( found )  return FD_FEC_RESOLVER_SHRED_IGNORED; /* With no packet loss, we expect found==1 about 50% of the time */

  set_ctx_t * ctx = ctx_map_query( curr_map, *w_sig, NULL );

  fd_bmtree_node_t leaf[1];
  uchar variant    = shred->variant;
  uchar shred_type = fd_shred_type( variant );

  if( FD_UNLIKELY( (shred_type==FD_SHRED_TYPE_LEGACY_DATA) | (shred_type==FD_SHRED_TYPE_LEGACY_CODE) ) ) {
    /* Reject any legacy shreds */
    return FD_FEC_RESOLVER_SHRED_REJECTED;
  }

  if( FD_UNLIKELY( shred->version!=resolver->expected_shred_version ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;

  int is_data_shred = fd_shred_is_data( shred_type );

  if( !is_data_shred ) { /* Roughly 50/50 branch */
    if( FD_UNLIKELY( (shred->code.data_cnt>FD_REEDSOL_DATA_SHREDS_MAX) | (shred->code.code_cnt>FD_REEDSOL_PARITY_SHREDS_MAX) ) )
      return FD_FEC_RESOLVER_SHRED_REJECTED;
    if( FD_UNLIKELY( (shred->code.data_cnt==0UL) | (shred->code.code_cnt==0UL) ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
  }

  ulong tree_depth           = fd_shred_merkle_cnt( variant ); /* In [0, 15] */
  ...
  ulong in_type_idx = fd_ulong_if( is_data_shred, shred->idx - shred->fec_set_idx, shred->code.idx );
  ulong shred_idx   = fd_ulong_if( is_data_shred, in_type_idx, in_type_idx + shred->code.data_cnt  );

  if( FD_UNLIKELY( in_type_idx >= fd_ulong_if( is_data_shred, FD_REEDSOL_DATA_SHREDS_MAX, FD_REEDSOL_PARITY_SHREDS_MAX ) ) )
    return FD_FEC_RESOLVER_SHRED_REJECTED;
  /* This, combined with the check on shred->code.data_cnt implies that
     shred_idx is in [0, DATA_SHREDS_MAX+PARITY_SHREDS_MAX). */

  if( FD_UNLIKELY( tree_depth>INCLUSION_PROOF_LAYERS-1UL             ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
  if( FD_UNLIKELY( fd_bmtree_depth( shred_idx+1UL ) > tree_depth+1UL ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;

  if( FD_UNLIKELY( !ctx ) ) {
    /* This is the first shred in the FEC set */
    ... 
    ...
    if( FD_UNLIKELY( FD_ED25519_SUCCESS != fd_ed25519_verify( _root->hash, 32UL, shred->signature, leader_pubkey, sha512 ) ) ) {
      freelist_push_head( free_list,        set_to_use );
      bmtrlist_push_head( bmtree_free_list, bmtree_mem );
      FD_MCNT_INC( SHRED, SHRED_REJECTED_INITIAL, 1UL );
      return FD_FEC_RESOLVER_SHRED_REJECTED;
    }

    /* This seems like a legitimate FEC set, so we can reserve some
       resources for it. */
    ctx = ctx_ll_insert( curr_ll_sentinel, ctx_map_insert( curr_map, *w_sig ) );
    ctx->set  = set_to_use;
    ctx->tree = tree;
    ctx->total_rx_shred_cnt = 0UL;
    ctx->data_variant   = fd_uchar_if(  is_data_shred, variant, fd_shred_variant( fd_shred_swap_type( shred_type ), (uchar)tree_depth ) );
    ctx->parity_variant = fd_uchar_if( !is_data_shred, variant, fd_shred_variant( fd_shred_swap_type( shred_type ), (uchar)tree_depth ) );

    ...
    /* Reset the FEC set */
    ctx->set->data_shred_cnt   = SHRED_CNT_NOT_SET;
    ctx->set->parity_shred_cnt = SHRED_CNT_NOT_SET;
    d_rcvd_join( d_rcvd_new( d_rcvd_delete( d_rcvd_leave( ctx->set->data_shred_rcvd   ) ) ) );
    p_rcvd_join( p_rcvd_new( p_rcvd_delete( p_rcvd_leave( ctx->set->parity_shred_rcvd ) ) ) );

  } else {
    /* This is not the first shred in the set */
    /* First, check to make sure this is not a duplicate */
    int shred_dup = fd_int_if( is_data_shred, d_rcvd_test( ctx->set->data_shred_rcvd,   in_type_idx ),
                                              p_rcvd_test( ctx->set->parity_shred_rcvd, in_type_idx ) );

    if( FD_UNLIKELY( shred_dup ) ) return FD_FEC_RESOLVER_SHRED_IGNORED;
    ...
  }

  /* Copy the shred to memory the FEC resolver owns */
  uchar * dst = fd_ptr_if( is_data_shred, ctx->set->data_shreds[ in_type_idx ], ctx->set->parity_shreds[ in_type_idx ] );
3.  fd_memcpy( dst, shred, shred_sz );

```

Note fd\_memcpy() on line #3.

We also need to see, how ctx->set->data\_shreds and ctx->set->parity\_shreds are allocated:

```
static void
unprivileged_init( fd_topo_t *      topo,
                   fd_topo_tile_t * tile,
                   void *           scratch ) {
 ...
 void * _fec_sets = FD_SCRATCH_ALLOC_APPEND( l, alignof(fd_fec_set_t),            sizeof(fd_fec_set_t)*fec_set_cnt   );

  fd_fec_set_t * fec_sets = (fd_fec_set_t *)_fec_sets;
  fd_shred34_t * shred34  = (fd_shred34_t *)store_out_dcache;

  for( ulong i=0UL; i<fec_set_cnt; i++ ) {
    fd_shred34_t * p34_base = shred34 + i*DCACHE_ENTRIES_PER_FEC_SET;
    for( ulong k=0UL; k<DCACHE_ENTRIES_PER_FEC_SET; k++ ) {
      fd_shred34_t * p34 = p34_base + k;

      p34->stride   = (ulong)p34->pkts[1].buffer - (ulong)p34->pkts[0].buffer;
      p34->offset   = (ulong)p34->pkts[0].buffer - (ulong)p34;
      p34->shred_sz = fd_ulong_if( k<2UL, 1203UL, 1228UL );
    }

    uchar ** data_shred   = fec_sets[ i ].data_shreds;
    uchar ** parity_shred = fec_sets[ i ].parity_shreds;
    for( ulong j=0UL; j<FD_REEDSOL_DATA_SHREDS_MAX;   j++ ) data_shred  [ j ] = p34_base[       j/34UL ].pkts[ j%34UL ].buffer;
    for( ulong j=0UL; j<FD_REEDSOL_PARITY_SHREDS_MAX; j++ ) parity_shred[ j ] = p34_base[ 2UL + j/34UL ].pkts[ j%34UL ].buffer;
  }

```

And we also need the declaration of fd\_shred34\_t structure:

```
struct __attribute__((aligned(FD_CHUNK_ALIGN))) fd_shred34 {
  ulong shred_cnt;
  ulong est_txn_cnt;
  ulong stride;
  ulong offset;
  ulong shred_sz; 
  union {
    fd_shred_t shred;
    uchar      buffer[ FD_SHRED_MAX_SZ ];
  } pkts[ 34 ];
};
typedef struct fd_shred34 fd_shred34_t;

```

So, ctx->set->data\_shreds are adjacent to each other in FEC set.

Thus fd\_memcpy() on line #3 will copy incoming shred to data\_shreds\[] array which is 1228 bytes in size.

If the size of incoming shred is larger than 1228, next shred in FEC set will be overwritten.

Also if shreds are coming out of order, that is - first shred comes with in\_type\_idx 1, than second shred with in\_type\_idx 0, it is possible to overwrite parts of first shred in FEC set.

Such overflow will invalidates first shred (already added) in FEC set, because it has been validated and its signature was checked before.

Note that Agave apparently discards such malformed shreds.

I see the following scenarios how it could be exploited:

1. slashing of FD node for producing bad blocks
2. consensus split between FD and Agave nodes as FD nodes will accept and parse such shreds, Agave will not
3. In case incoming shred is the last shred in pkts\[] array, heap overflow will occur.

This could be potentially be a promising remote code execution vulnerability, as shreds are coming from network.

Currently, looke like RCE vector is not possible to exploit, as fd\_shred34 structure lays in the middle of huge mapped region of 3GB in size.

## Impact Details

Consensus split between FD and Agave nodes. Possibility of RCE.

## Proof of concept

## Proof of Concept

How to reproduce:

1. get archive by using provided gist link
2. unpack it:

```
$ base64 -d arch.txt > arch.tgz
$ tar zxf arch.tgz

```

3. copy provided test\_fec\_resolver.c over src/disco/shred/test\_fec\_resolver.c
4. build FD with:

```
EXTRAS="asan" make -j unit-test
```

5. run test\_fec\_resolver unit-test:

```
$ ...test_fec_resolver test1.bin

=================================================================
==162381==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x61a00000054c at pc 0x562f4677584a bp 0x7ffde97060b0 sp 0x7ffde9705880
WRITE of size 1648 at 0x61a00000054c thread T0
    #0 0x562f46775849 in __asan_memcpy (/build/linux/clang/x86_64/unit-test/test_fec_resolver+0xb6849) (BuildId: 741307849f3df20bb7c98e537e880c65c37056cd)
    #1 0x562f467b8b80 in memcpy /usr/include/x86_64-linux-gnu/bits/string_fortified.h:29:10
    #2 0x562f467b8b80 in fd_memcpy /src/disco/shred/../../ballet/shred/../bmtree/../../util/fd_util_base.h:1011:10
    #3 0x562f467b8b80 in fd_fec_resolver_add_shred /src/disco/shred/fd_fec_resolver.c:519:2
    #4 0x562f467b43c8 in test_one_batch /src/disco/shred/test_fec_resolver.c:106:8
    #5 0x562f467b43c8 in main /src/disco/shred/test_fec_resolver.c:135:4
    #6 0x710c23e29d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
    #7 0x710c23e29e3f in __libc_start_main csu/../csu/libc-start.c:392:3
    #8 0x562f466f3624 in _start (/build/linux/clang/x86_64/unit-test/test_fec_resolver+0x34624) (BuildId: 741307849f3df20bb7c98e537e880c65c37056cd)
    ...
SUMMARY: AddressSanitizer: heap-buffer-overflow (/build/linux/clang/x86_64/unit-test/test_fec_resolver+0xb6849) (BuildId: 741307849f3df20bb7c98e537e880c65c37056cd) in __asan_memcpy

```

6. proof of concept script t1.py should be tested against live FD, but before we need a few modifictions to the code (to simplify the testing):

6.1) comment out lines 548-552 <https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/run/tiles/fd\\_shred.c#L548>

6.2) comment out lines 439-442 <https://github.com/firedancer-io/firedancer/blob/main/src/disco/shred/fd\\_fec\\_resolver.c#L439>

6.3) after fd\_memcpy() <https://github.com/firedancer-io/firedancer/blob/main/src/disco/shred/fd\\_fec\\_resolver.c#L497> add the following code (we are checking if next shred in FEC set has been overwritten):

```
 ulong *ptr = (ulong * ) ctx->set->data_shreds[in_type_idx + 1];
 FD_TEST( *ptr != 0x7878787878787878);
```

7. run FD:

```
#./build/linux/clang/x86_64/bin/fdctl run --config config.toml

```

8. run t1.py:

```
$ ./t1.py host
```

9. notice that shred tile crashes with message, which means adjacent shred in FEC set has been overwritten:

```
 ... shred:0 src/disco/shred/fd_fec_resolver.c(521): FAIL: *ptr != 0x7878787878787878

```
