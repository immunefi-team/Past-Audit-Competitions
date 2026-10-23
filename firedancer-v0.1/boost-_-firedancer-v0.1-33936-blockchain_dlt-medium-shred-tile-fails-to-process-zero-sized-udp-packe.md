# Boost \_ Firedancer v0.1 33936 - \[Blockchain\_DLT - Medium] shred tile fails to process zero sized udp

Submitted on Fri Aug 02 2024 02:05:48 GMT-0400 (Atlantic Standard Time) by @gln for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33936

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Any bug leading to loss of funds or acceptance of forged / invalid signatures

## Description

## Brief/Intro

There is an issue in fd\_shred.c, when it tries to process packets from net tile - if it sees a zero-sized UDP packet it will call abort().

## Vulnerability Details

There is a net\_shred link, meaning that shred tile can process data from net tile.

To do so shred tile calls during\_frag() function <https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/run/tiles/fd\\_shred.c#L297>

Let's look at the code:

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

  ctx->tsorig = fd_frag_meta_ts_comp( fd_tickcount());
  ...
  ...
  } else { /* the common case, from the netmux tile */
    if( FD_UNLIKELY( chunk<ctx->net_in_chunk0 || chunk>ctx->net_in_wmark || sz>FD_NET_MTU ) )
      FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->net_in_chunk0, ctx->net_in_wmark ));
    uchar const * dcache_entry = fd_chunk_to_laddr_const( ctx->net_in_mem, chunk );
1.    ulong hdr_sz = fd_disco_netmux_sig_hdr_sz( sig );
2.    FD_TEST( hdr_sz < sz ); /* Should be ensured by the net tile */
    fd_shred_t const * shred = fd_shred_parse( dcache_entry+hdr_sz, sz-hdr_sz );
    if( FD_UNLIKELY( !shred ) ) {
      *opt_filter = 1;
      return;
    };
    /* all shreds in the same FEC set will have the same signature
       so we can round-robin shreds between the shred tiles based on
       just the signature without splitting individual FEC sets. */
    ulong sig = fd_ulong_load_8( shred->signature );
    if( FD_LIKELY( sig%ctx->round_robin_cnt!=ctx->round_robin_id ) ) {
      *opt_filter = 1;
      return;
    }
    fd_memcpy( ctx->shred_buffer, dcache_entry+hdr_sz, sz-hdr_sz );
    ctx->shred_buffer_sz = sz-hdr_sz;
  }
}
```

What we are interested here are these 2 lines:

1. header size is pulled from frame signature
2. it is compared against packet size by using FD\_TEST macro

Let's look at the definition of FD\_TEST macro:

```
#define FD_LOG_ERR(a)             do { long _fd_log_msg_now = fd_log_wallclock(); fd_log_private_2( 4, _fd_log_msg_now, __FILE__, __LINE__, __func__, fd_log_private_0           a ); } while(0)

#define FD_TEST(c) do { if( FD_UNLIKELY( !(c) ) ) FD_LOG_ERR(( "FAIL: %s", #c )); } while(0)

void
fd_log_private_2( int          level,
                  long         now,
                  char const * file,
                  int          line,
                  char const * func,
                  char const * msg ) {
  fd_log_private_1( level, now, file, line, func, msg );

# if FD_LOG_UNCLEAN_EXIT && defined(__linux__)
  if( level<fd_log_level_core() ) syscall(SYS_exit_group, 1);
# else
  if( level<fd_log_level_core() ) exit(1); /* atexit will call fd_log_private_cleanup implicitly */
# endif

  abort();
}

```

Thus, if hdr\_sz is equal to sz, which is the case when UDP packet with no data arrives, shred tile eventually calls abort() and exists.

As a result firedancer will stop working.

## Impact Details

Attacker will be able to crash firedancer remotely.

## Proof of concept

## Proof of Concept

How to reproduce:

1. get firedancer source
2. change fd\_net.c , after line <https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/run/tiles/fd\\_net.c#L259> , add the following code:

```
ulong hdr_sz = fd_disco_netmux_sig_hdr_sz(sig);
FD_TEST(hdr_sz < batch[i].buf_sz);

```

It simulates the FD\_TEST call of shred tile.

3. edit fd\_mux.c, comment out line #166 <https://github.com/firedancer-io/firedancer/blob/main/src/disco/mux/fd\\_mux.c#L166>
4. edit config.c, change these calls to FD\_LOG\_ERR to FD\_LOG\_WARNING:

<https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/config.c#L546>

<https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/config.c#L653>

After that you have to build firedancer with 'make -j fdctl'

5. run net tile, config.toml can be found here - <https://gist.github.com/gln7/d0a699b3e09d52cdc2881705e8983378>

```
# ./build/linux/clang/x86_64/bin/fdctl run1 net 0 --config config.toml
WARNING XX 582875 f0   0    src/app/fdctl/config.c(549): Trying to use [gossip.host] 192.168.123.29 for listening to incoming transactions, but it is part of a private network and will not be routable for other Solana network nodes.
WARNING XX 582875 f0   0    src/app/fdctl/config.c(653): trying to join a live cluster, but configuration disables the sandbox which is a a development only feature
Log at "/tmp/fd-0.0.0_582875_xx"
NOTICE  xx 582877 f0   net:0 src/disco/topo/fd_topo_run.c(32): booting tile net:0 pid:582875 tid:582877


```

6. download t1.py by using gist link and run it

```
$./t1.py 192.168.123.29
```

7. net tile exits:

```
ERR xx 582877 f0   net:0 src/app/fdctl/run/tiles/fd_net.c(271): FAIL: hdr_sz < batch[i].buf_sz

```
