# Boost \_ Firedancer v0.1 34064 - \[Blockchain\_DLT - Medium] bank tile possible code execution

Submitted on Mon Aug 05 2024 02:31:39 GMT-0400 (Atlantic Standard Time) by @gln for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #34064

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Any sandbox escape

## Description

## Brief/Intro

Bank and poh tiles do not have checks for minimum fragment size.

Also bank tile trusts the pointer it receives in fragment from pack tile.

## Vulnerability Details

Several tiles do not have lower bound checks for size of incoming fragments.

Let's look at the code:

1. poh tile <https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/run/tiles/fd\\_poh.c#L1392>

```
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

  fd_poh_ctx_t * ctx = (fd_poh_ctx_t *)_ctx;
  ...
  if( FD_UNLIKELY( chunk<ctx->bank_in[ in_idx ].chunk0 || chunk>ctx->bank_in[ in_idx ].wmark || sz>USHORT_MAX ) )
      FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->bank_in[ in_idx ].chunk0, ctx->bank_in[ in_idx ].wmark ));

    uchar * src = (uchar *)fd_chunk_to_laddr( ctx->bank_in[ in_idx ].mem, chunk );

    fd_memcpy( ctx->_txns, src, sz-sizeof(fd_microblock_trailer_t) );
    fd_memcpy( ctx->_microblock_trailer, src+sz-sizeof(fd_microblock_trailer_t), sizeof(fd_microblock_trailer_t) );

```

2. bank tile <https://github.com/firedancer-io/firedancer/blob/main/src/app/fdctl/run/tiles/fd\\_bank.c#L107>

```
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

1.  if( FD_UNLIKELY( chunk<ctx->pack_in_chunk0 || chunk>ctx->pack_in_wmark || sz>USHORT_MAX ) )
    FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->pack_in_chunk0, ctx->pack_in_wmark ));

2.  fd_memcpy( dst, src, sz-sizeof(fd_microblock_bank_trailer_t) );
  fd_microblock_bank_trailer_t * trailer = (fd_microblock_bank_trailer_t *)( src+sz-sizeof(fd_microblock_bank_trailer_t) );
3.  ctx->_bank = trailer->bank;
}
```

1. there is an upper bound check for 'sz'
2. in case 'sz' is less than sizeof(fd\_microblock\_bank\_trailer\_t) memcpy with very large length will be called on this line

This could possibly lead to code execution in poh/bank tiles and sandbox escape, because both of these tiles run in agave process as threads and basically have no sandbox.

3. Another interesting issue is that fd\_bank tile does not have any checks for 'bank' pointer.

It is being initialized from incoming fragment, see line #3 on the above code snippet.

Later on this pointer is passed to fd\_ext\_\_ rust calls:

```
pub extern "C" fn fd_ext_bank_pre_balance_info( bank: *const std::ffi::c_void, txns: *const std::ffi::c_void,
 txn_count: u64 ) -> *mut std::ffi::c_void {
    use solana_sdk::transaction::SanitizedTransaction;
    use std::borrow::Cow;
    use std::sync::atomic::Ordering;

    let txns = unsafe {
        std::slice::from_raw_parts(txns as *const SanitizedTransaction, txn_count as usize)
    };
    let bank = bank as *const Bank;
1.    unsafe { Arc::increment_strong_count(bank) };
    let bank = unsafe { Arc::from_raw( bank as *const Bank ) };
     ...
}
```

1. what happens here is that Arc::increment\_strong\_count() increments refcount by using our bank pointer.

This issue gives us an ability to increment the value at any address in memory, which is quite powerful primitive.

It could lead to code execution in bank tile/agave process and escape out of sandbox.

## Impact Details

Remote code execution in poh/bank tiles. Sandbox escape as these tiles are not sandboxed.

## Proof of concept

## Proof of Concept

How to reproduce:

1. get firedancer source code
2. to simplify the testing I've slightly modified the code
3. src/app/fdctl/config.c on lines 546, 551, 653: change FD\_LOG\_ERR macro to FD\_LOG\_INFO
4. disco/mux/fd\_mux.c - comment out line 166
5. fd\_bank.c - change line 129 to something like " ctx->\_bank = trailer->bank;ctx->\_bank=(void\*)0x4142434451525354;sleep(20);". Make sure to add "#include \<unistd.h>" at the beginning as well.
6. disco/mux/fd\_mux.c - comment out line 608 (do not call 'continue')
7. To test memcpy() issue: disco/mux/fd\_mux.c - add code on line 643 "chunk=32841;sz=4;"
8. build firedancer with asan
9. run bank tile:

```

# ./build/linux/clang/x86_64/bin/fdctl run1 bank 0 --config config.toml
...
=================================================================
==821897==ERROR: AddressSanitizer: negative-size-param: (size=-8)
==821899==WARNING: external symbolizer didn't start up correctly!
==821900==WARNING: external symbolizer didn't start up correctly!
==821901==WARNING: external symbolizer didn't start up correctly!
==821902==WARNING: external symbolizer didn't start up correctly!
==821903==WARNING: external symbolizer didn't start up correctly!
==821903==WARNING: Failed to use and restart external symbolizer!
    #0 0x5a4b85f5db14  (firedancer/build/linux/clang/x86_64/bin/fdctl+0xb26b14) (BuildId: e995404f26f63a7b7035fcf82d3fde62f5930048)
    #1 0x5a4b86003e88  (firedancer/build/linux/clang/x86_64/bin/fdctl+0xbcce88) (BuildId: e995404f26f63a7b7035fcf82d3fde62f5930048)
    #2 0x5a4b86061120  (firedancer/build/linux/clang/x86_64/bin/fdctl+0xc2a120) (BuildId: e995404f26f63a7b7035fcf82d3fde62f5930048)
    #3 0x5a4b8605d0fc  (firedancer/build/linux/clang/x86_64/bin/fdctl+0xc260fc) (BuildId: e995404f26f63a7b7035fcf82d3fde62f5930048)
    #4 0x5a4b85fc0b17  (firedancer/build/linux/clang/x86_64/bin/fdctl+0xb89b17) (BuildId: e995404f26f63a7b7035fcf82d3fde62f5930048)
    #5 0x73e5d9d25a03  (/lib/x86_64-linux-gnu/libc.so.6+0x125a03) (BuildId: 490fef8403240c91833978d494d39e537409b92e)

 
```

10. To test 'bank' pointer issue, edit fd\_mux.c file on line 643 , change it to "chunk=32841;sz=20;"
11. run bank tile, it will fork and go to background:

```
# ./build/linux/clang/x86_64/bin/fdctl run1 bank 0 --config config.toml

```

12. get the pid of 'fdctl' process - "ps aux|grep fdctl" and attach to it by using gdb, enter "c" and press enter
13. after some time bank tile will crash:

```
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x00007e6418ce578a in __GI___clock_nanosleep (clock_id=clock_id@entry=0, flags=flags@entry=0, req=req@entry=0x2454dc1fc800, rem=rem@entry=0x2454dc1fc800) at ../sysdeps/unix/sysv/linux/clock_nanosleep.c:78
78	../sysdeps/unix/sysv/linux/clock_nanosleep.c: No such file or directory.
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
fd_ext_bank_pre_balance_info () at /rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library/core/src/sync/atomic.rs:3321
3321	/rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library/core/src/sync/atomic.rs: No such file or directory.
(gdb) bt
#0  fd_ext_bank_pre_balance_info ()
    at /rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library/core/src/sync/atomic.rs:3321
#1  0x00005b895f7eb1c5 in after_frag (_ctx=0x35b75a801000, in_idx=<optimized out>, 
    seq=18446744073709551615, opt_sig=0x2454dc1fdd60, opt_chunk=<optimized out>, opt_sz=<optimized out>, 
    opt_tsorig=0x2454dc1fdde0, opt_filter=0x2454dc1fddb0, mux=0x2454dc1fdd00)
    at src/app/fdctl/run/tiles/fd_bank.c:205
#2  0x00005b895f84822b in fd_mux_tile (cnc=cnc@entry=0x29f7fe404600, flags=flags@entry=3, 
    in_cnt=in_cnt@entry=1, in_mcache=in_mcache@entry=0x2454dc1fe600, in_fseq=in_fseq@entry=0x2454dc1fee80, 
    mcache=mcache@entry=0x28d300001100, out_cnt=1, _out_fseq=0x2454dc1ff300, burst=1, cr_max=128, 
    lazy=<optimized out>, rng=0x2454dc1ffbe0, scratch=0x2454dc1fdf80, ctx=0x35b75a801000, 
    callbacks=0x2454dc1ffb80) at src/disco/mux/fd_mux.c:663
#3  0x00005b895f8440fd in fd_topo_run_tile (topo=<optimized out>, tile=<optimized out>, 
    sandbox=<optimized out>, uid=<optimized out>, gid=<optimized out>, allow_fd=<optimized out>, 
    wait=<optimized out>, debugger=<optimized out>, tile_run=<optimized out>)
    at src/disco/topo/fd_topo_run.c:171
#4  0x00005b895f7a7b18 in tile_main (_args=<optimized out>) at src/app/fdctl/run/run1.c:53
#5  0x00007e6418d25a04 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:100
..
(gdb) x/2i $pc
=> 0x5b895ff5cdd4 <fd_ext_bank_pre_balance_info+20>:	lock incq -0x10(%rdi)
   0x5b895ff5cdd9 <fd_ext_bank_pre_balance_info+25>:	
    jle    0x5b895ff5d19c <fd_ext_bank_pre_balance_info+988>
(gdb) i r rdi
rdi            0x4142434451525354  4702394921629406036
(gdb) 

```

As you can see, basically it is increment-anywhere primitive
