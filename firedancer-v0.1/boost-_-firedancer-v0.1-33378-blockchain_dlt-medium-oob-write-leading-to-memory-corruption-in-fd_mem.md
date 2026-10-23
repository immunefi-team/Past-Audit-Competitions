# Boost \_ Firedancer v0.1 33378 - \[Blockchain\_DLT - Medium] OOB Write leading to memory corruption in

Submitted on Thu Jul 18 2024 23:33:45 GMT-0400 (Atlantic Standard Time) by @c4a4dda89 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33378

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

In the `during_frag` function within the fd\_sign tile implementation (src/app/fdctl/run/tiles/fd\_sign.c), The "sz" parameter is not subject to any boundary checks, which can lead to an Out-Of-Bounds (OOB) Write when an attacker controls the "sz". In the case `FD_KEYGUARD_ROLE_VOTER`

```c
static void FD_FN_SENSITIVE
during_frag_sensitive( void * _ctx,
                       ulong  in_idx,
                       ulong  seq,
                       ulong  sig,
                       ulong  chunk,
                       ulong  sz,
                       int *  opt_filter ) {
  (void)seq;
  (void)sig;
  (void)chunk;
  (void)sz;
  (void)opt_filter;

  fd_sign_ctx_t * ctx = (fd_sign_ctx_t *)_ctx;
  FD_TEST( in_idx<MAX_IN );

  switch( ctx->in_role[ in_idx ] ) {
    case FD_KEYGUARD_ROLE_VOTER: // no check here
      fd_memcpy( ctx->_data, ctx->in_data[ in_idx ], sz );
      break;
    case FD_KEYGUARD_ROLE_LEADER:
      fd_memcpy( ctx->_data, ctx->in_data[ in_idx ], 32UL );
      break;
    case FD_KEYGUARD_ROLE_TLS:
      fd_memcpy( ctx->_data, ctx->in_data[ in_idx ], 130UL );
      break;
    case FD_KEYGUARD_ROLE_GOSSIP:
      if( sz>FD_KEYGUARD_SIGN_REQ_MTU ) {
        FD_LOG_WARNING(("Corrupt gossip signing message with size %lu", sz));
        *opt_filter = 1;
        return;
      }
      fd_memcpy( ctx->_data, ctx->in_data[ in_idx ], sz );
      break;
    case FD_KEYGUARD_ROLE_REPAIR:
      if( sz>FD_KEYGUARD_SIGN_REQ_MTU ) {
        FD_LOG_WARNING(("Corrupt repair signing message with size %lu", sz));
        *opt_filter = 1;
        return;
      }
      fd_memcpy( ctx->_data, ctx->in_data[ in_idx ], sz );
      break;
    default:
      FD_LOG_CRIT(( "unexpected link role %lu", ctx->in_role[ in_idx ] ));
  }
}
```

## Vulnerability Details

```c
case FD_KEYGUARD_ROLE_VOTER:
      fd_memcpy( ctx->_data, ctx->in_data[ in_idx ], sz );
      break;
```

The "sz" parameter is not subject to any boundary checks, which can lead to an Out-Of-Bounds (OOB) Write when an attacker controls the "sz".

## Impact Details

Process-to-process memory corruption may lead to the process-to-process RCE between sandboxed tiles.

## References

<https://github.com/firedancer-io/firedancer/blob/e60d9a6206efaceac65a5a2c3a9e387a79d1d096/src/app/fdctl/run/tiles/fd\\_sign.c#L78>

## Proof of concept

## Proof of Concept

The lack of a PoC (Proof of Concept) stems from the complexity of the attack. Since it involves a process-to-process exploit, modifying the producer to generate untrusted input is necessary. We highly encourage the Firedancer team to review the report for a comprehensive explanation of the vulnerability. The report includes code snippets that clearly demonstrate the issue.
