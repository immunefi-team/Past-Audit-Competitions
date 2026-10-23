# #40530 \[W\&A-High] JWT Salt Expiration isn't entirely correct in wallet\_rpc\_server::auth\_http\_request

**Submitted on Feb 26th 2025 at 01:32:59 UTC by @jovi for** [**IOP | Zano**](https://immunefi.com/audit-competition/iop-zano)

* **Report ID:** #40530
* **Report Type:** Websites and Applications
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/zano-iop/blob/main/src/wallet/wallet\\_rpc\\_server.cpp>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.
  * Malicious interactions with an already-connected wallet, such as:
* Modifying transaction arguments or parameters
* Substituting contract addresses
* Submitting malicious transactions

## Description

## Summary

A logic flaw in how the server calculates salt expiration allows replay of JWT tokens in Zano's Wallet RPC. Specifically, the code uses `epee::misc_utils::get_tick_count()`—which returns milliseconds—to determine `ticks_now`, but the salt’s retention duration (`JWT_TOKEN_EXPIRATION_MAXIMUM`) is defined in seconds. This mismatch causes the server to “forget” salts early, re-enabling replayed tokens.

***

## Vulnerability Details

1. **Location**
   * `wallet_rpc_server.cpp`, function `bool wallet_rpc_server::auth_http_request(...)`.
   * The relevant time function is defined in `epee::misc_utils::get_tick_count()`, which derives its return value in milliseconds:

     ```cpp
     inline uint64_t get_tick_count()
     {
     #if defined(_MSC_VER)
       // ...
     #elif defined(__MACH__)
       // ...
     #else
       struct timespec ts;
       if(clock_gettime(CLOCK_MONOTONIC, &ts) != 0) {
               return 0;
       }
       return (ts.tv_sec * 1000) + (ts.tv_nsec/1000000);
     #endif
     }
     ```
   * In `wallet_rpc_server::auth_http_request`, that ticks value is read into `ticks_now`:

     ```cpp
     uint64_t ticks_now = epee::misc_utils::get_tick_count();
     m_jwt_used_salts.add(salt, ticks_now + JWT_TOKEN_EXPIRATION_MAXIMUM);
     m_jwt_used_salts.remove_if_expiration_less_than(ticks_now);
     ```
   * Because `JWT_TOKEN_EXPIRATION_MAXIMUM` is `60 * 60` (seconds) instead of `60 * 60 * 1000` (milliseconds), salts are purged prematurely.
2. **Description**
   * The server stores each token’s `salt` for a certain “expiration” window to prevent replay attacks (i.e., a second use of the same token).
   * If `JWT_TOKEN_EXPIRATION_MAXIMUM` is 3600 and `get_tick_count()` is in milliseconds, the code will only keep salts for \~3.6 seconds (instead of one hour).
   * After the server’s next request-based cleanup, a just-used token’s salt is dropped from memory far too soon, inadvertently allowing the token to be reused.
3. **Code Snippets**\
   \&#xNAN;**`wallet_rpc_server::auth_http_request`:**

   ```cpp
   bool wallet_rpc_server::auth_http_request(...)
   {
     // ...
     uint64_t ticks_now = epee::misc_utils::get_tick_count();
     // @audit j-05 -> if ticks_now is in ms, JWT_TOKEN_EXPIRATION_MAXIMUM is a weak cooldown ( 60 * 60)
     m_jwt_used_salts.add(salt, ticks_now + JWT_TOKEN_EXPIRATION_MAXIMUM);
     m_jwt_used_salts.remove_if_expiration_less_than(ticks_now);
     // ...
   }
   ```

   **`epee::misc_utils::get_tick_count`:**

   ```cpp
   inline uint64_t get_tick_count()
   {
   #if defined(_MSC_VER)
     // ...
   #elif defined(__MACH__)
     // ...
   #else
     struct timespec ts;
     if(clock_gettime(CLOCK_MONOTONIC, &ts) != 0) {
       return 0;
     }
     return (ts.tv_sec * 1000) + (ts.tv_nsec / 1000000);
   #endif
   }
   ```

**4. Impact**

* **Operating Context**
  * By **default**, the wallet RPC server is typically run locally (bound to `127.0.0.1`), which limits exposure under normal configurations. However, **if** it is **exposed** to an untrusted network—e.g., on a public interface or open Wi-Fi—this vulnerability becomes much more dangerous.
* **Attack Complexity**
  * An attacker needs to **sniff** the unencrypted (HTTP) traffic and intercept a valid JWT token. No special privileges or direct code execution is required—just local network access or the ability to position oneself on‐path.
* **High Potential Impact**
  1. **Replay Attacks**:
     * Due to the **premature salt removal**, JWT tokens become valid again after \~3.6 seconds. The attacker can resend them indefinitely, bypassing replay protection entirely.
     * Unauthorized parties can use replayed tokens to **invoke protected RPC endpoints** (e.g., transfer funds).
     * While the server’s secret key itself is not disclosed, replay defeats the intended “one‐time use” property of the token.
  2. **Complete Wallet Control**:
     * Because the RPC enables critical operations (transfers, address management, and more), replaying a stolen token effectively grants **full control** of the wallet.
     * Funds can be stolen or manipulated **without** any further user interaction.

## Proof of Concept

1. **Intercept** a valid JWT from a legitimate request to the Wallet RPC.
2. **Wait** for about 4–5 seconds (or until any subsequent request triggers salt cleanup).
3. **Replay** the same token: the server sees no record of the salt (purged) and permits the exact same request again.

Especifically for this program: *runnable PoC code is not required. Whitehats are instead required to write a step-by-step explanation of the PoC and impact.*
