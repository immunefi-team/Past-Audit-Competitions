# #39993 \[W\&A-Low] node-fetch without response limit

**Submitted on Feb 12th 2025 at 12:01:18 UTC by @riproprip for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39993
* **Report Type:** Websites and Applications
* **Report severity:** Low
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The archiver uses `node-fetch` without response size limits. This allows attackers to use up all the memory on the system until it crashes.

## Vulnerability Details

`node-fetch` should be called with a response limit. Since a limit is not supplied and `node-fetch` allows compressed responses, the counterparty can return a bunch of compressed stuff to use very little bandwith/transfer volume/resources to crash a node.

Please note that while the attacker does not initiate the attack itself, the victim has to make contact with other archivers/nodes to function (giving attackers the option to strike).

## Impact Details

Usually just the node process gets killed. In rare cases the OS also kills other processes.

## References

This happens on multiple occasions throughout the code. A search for "fetch(" probably finds all the instances.

\[1] Youtube video demonstrating on a 16GB memory system. `https://youtu.be/cGheyyusKXc`

## Notes

Since this bug is similar in nature to `39395` it makes sense to go through triage with that one first. Before resolving this bug.

## Proof of Concept

## Proof of Concept

The POC is very similar to: 39395

### steps

```
apt-get update
apt-get -y install git-core curl build-essential python3 vim 

wget -O "/tmp/compressed_34.json" "https://gist.githubusercontent.com/cki/74224edfa2ba4dd2cdf78e4f229bcc38/raw/a14c7f76032f84ea0af2ae63c0e94df693611e80/gistfile1.txt"
wget -O "/tmp/shardeum-archiver-poc.patch" "https://gist.githubusercontent.com/cki/3e698c1a305a888fdc8638085314c951/raw/8b6812c9aec66f7da4eff21a911b3b8f37ac160b/shardeum-archiver-poc.patch"

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash > /dev/null
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
nvm install 18.19.1 2> /dev/null
nvm use 18.19.1 2> /dev/null
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup install 1.74.1
rustup default 1.74.1

git clone https://github.com/shardeum/shardeum.git
cd shardeum && git checkout tags/bugbounty
nvm use 18.19.1
git apply debug-10-nodes.patch ( keeping the mode in release ) 
# manually edit src/config/index.ts -> flexibleRotationDelta: 0
npm ci ;
npm run prepare ; 
npm install -g shardus ;
npm update @shardus/archiver ;

git apply /tmp/shardeum-archiver-poc.patch

free -m
shardus start 1 pm2--no-autorestart # start one node and one archiver
shardus start 1 -a 1 pm2--no-autorestart # start a second archiver
```

### notes

Should your node have more than 16GB of RAM please use following link for the compressed\_34.json. It works up till 256GB of RAM.

`https://gist.githubusercontent.com/cki/bcc3d28f00c57db78b8916dcc0386462/raw/87f582828ebcb645207b5f1d546dc4b072ab017f/256gb_compressed.json`
