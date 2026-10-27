# Boost \_ Firedancer v0.1 33347 - \[Blockchain\_DLT - Medium] Integer underflow leading to memory corrup

Submitted on Thu Jul 18 2024 10:42:07 GMT-0400 (Atlantic Standard Time) by @c4a4dda89 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33347

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Process to process Memory Corruption between sandboxed tiles (may lead to code execution)

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

In the `during_frag` function within the fd\_bank tile implementation (src/app/fdctl/run/tiles/fd\_bank.c), there is insufficient validation of the sz field. The function only verifies that sz is less than `USHRT_MAX`, but it does not check for a lower limit.

```c
  if( FD_UNLIKELY( chunk<ctx->pack_in_chunk0 || chunk>ctx->pack_in_wmark || sz>USHORT_MAX ) )
    FD_LOG_ERR(( "chunk %lu %lu corrupt, not in range [%lu,%lu]", chunk, sz, ctx->pack_in_chunk0, ctx->pack_in_wmark ));
```

Specifically, consider the line:

`fd_memcpy( dst, src, sz-sizeof(fd_microblock_bank_trailer_t) );`

If sz is less than the size of fd\_microblock\_bank\_trailer\_t, attempting to subtract its size will result in an integer underflow. This can lead to memory corruption, potentially allowing an attacker to compromise the target tile.

## Vulnerability Details

In the definition of `fd_mux_during_frag_fn`, it’s clearly described that the inputs are not trusted because the producer could put arbitrary values in a frag.

during\_frag in fd\_bank.c

```c
fd_memcpy( dst, src, sz-sizeof(fd_microblock_bank_trailer_t) );
```

## Impact Details

Process-to-process memory corruption may lead to the process-to-process RCE between sandboxed tiles.

## References

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_bank.c#L123>

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_bank.c#L126>

## Proof of concept

## Proof of Concept
