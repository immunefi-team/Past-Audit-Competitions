# 26520 - \[SC - Insight] Multisig Contract onChain can be bricked

Submitted on Dec 4th 2023 at 12:26:36 UTC by @copperscrewer for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26520

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

The Multisig contract, on a series of actions can be bricked. As it cannot execute anymore actions and fails to satisfy an Invariant ( length of owners is equal or greater than required signatures)

## Impact

Multisig cannot execute any transaction, cannot add, remove Owners as the contract becomes unusable

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:6.9

## Recommendation

Add a notNull modifier to replaceOwner

## Proof of concept

steps to reproduce As we are testing the Gnosis Multisig, that works independently to the L2 contracts, we can test just this contract while replicating the Owners and required state.

Create an empty folder.

npx hardhat init

To create a project with directories like contracts, test

copy paste contract code from <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52?utm\\_source=immunefi#code>

to gnosis.sol in contracts folder

change compiler version or directly paste this in module exports in hardhat config.js This is done since the Gnosis multisig contract uses an old version of solidity solidity: { compilers: \[ { version: "0.4.22", }, { version: "0.8.19", settings: {}, }, ], },

paste following test script in test folder and run npx hardhat test test/gnosis.js\\

```
const {ethers} = require("hardhat")
const {expect, assert} = require("chai")

var gnosis

let owners, deployer

describe("ReplaceOwner bug", function()
{
    before(async function(){

        owners = (await ethers.getSigners()).slice(0,6)
        deployer = owners[0]
        
    })

    context("Setup Contract (4/6 multisig) ", () =>    {

        it("Should deploy with 6 owners, 4 required ", async function(){

            let gnosisF = await ethers.getContractFactory("MultiSigWallet", deployer)
            gnosis = await gnosisF.deploy(
                owners,
                4
            );

            expect(await gnosis.required()).to.eq(4);
            expect( (await gnosis.getOwners() ).length).to.eq(6);

        })

    })

    context("Bug proof of concept", () => {
        
        it("Should remove two owners", async function(){

            let ABI = [
                "function transfer(address to, uint amount)",
                "function removeOwner(address owner)"
            ];
    
            let iface = new ethers.Interface(ABI);
            data = iface.encodeFunctionData("removeOwner", [ owners[owners.length - 1].address, ] )

            //one owner proposes an action through submitTransaction and the others
            //adhere through confirmTransaction

            // remove one owner [CODE START]
            await gnosis.submitTransaction(
                await gnosis.getAddress(), //destination
                0, //value
                data //data
                )
            
            for( i=1; i<=3; i++)
            {
                
                signer = owners[i];
                await gnosis.connect(signer).confirmTransaction(
                    0 //transaction id
                );
                
            }
            
            
            expect((await gnosis.getOwners()).length).to.eq(5)
            // remove one owner [CODE END]

            // remove another owner [CODE START]
            data = iface.encodeFunctionData("removeOwner", [ owners[owners.length - 2].address, ] )
            await gnosis.submitTransaction(
                await gnosis.getAddress(), //destination
                0, //value
                data //data
                )
            
            for( i=1; i<=3; i++)
            {
                
                signer = owners[i];
                await gnosis.connect(signer).confirmTransaction(
                    1 //transaction id
                );
                
            }
            
            
            expect((await gnosis.getOwners()).length).to.eq(4)
            // remove another owner [CODE END]
            
            
        })

        it("Should replace an Owner with zero address", async function(){
            let ABI = [
                "function transfer(address to, uint amount)",
                "function replaceOwner(address owner, address newOwner)"
            ];
    
            let iface = new ethers.Interface(ABI);
            
            data = iface.encodeFunctionData("replaceOwner", [ owners[2].address, '0x0000000000000000000000000000000000000000'] )
            await gnosis.submitTransaction(
                await gnosis.getAddress(), //destination
                0, //value
                data //data
                )
            
            for( i=1; i<=3; i++)
            {                
                signer = owners[i];
                await gnosis.connect(signer).confirmTransaction(
                    2 //transaction id
                );
                
            }
            

            expect(await gnosis.required()).to.eq(4);
            expect((await gnosis.getOwners()).length).to.eq(4);          

            console.log("Owners are ", await gnosis.getOwners() );
            console.log("Required amount of signatures are ", await gnosis.required() );
        })

    })

})
```
