---
id: basil
title: Zeppelin's Basil
sidebar_label: Basil
---

Here at the Zeppelin headquarters we have a basil plant. She is a good mascot, always green, always faithful. For reasons unknown, we found that she enjoys a lot being under a light that changes color; so of course we got her the best multicolor LED bulb we could find.

![The Basil](https://pbs.twimg.com/media/DdL2qciX4AEMeoR.jpg "The basil")

However, after a few days we started having conflicts. Who gets the honor to set the light color for our friendly plant? What if they choose their favorite color instead of the one that's best for the plant? For how long do they get to keep their chosen color? We also found that somebody kept resetting the color back to an ugly lime green every morning. We are ok with anarchy, but we want transparency, so we decided to control the light bulb through a contract on the Ethereum blockchain.

## Creating an app with ZeppelinOS

In this guide, we will build a simple dApp on top of ZeppelinOS. To see the end product, please visit:
* Source code: [zeppelinos/basil](https://github.com/zeppelinos/basil)
* App: [basil.zeppelin.solutions](https://basil.zeppelin.solutions)

> NOTE: You will see several Ethereum addresses output by the commands you execute as you follow along with this tutorial.  Do not be concerned if they don't match the addresses you see in this document.

First we will need to [install Node.js following the instructions from their website](https://nodejs.org/en/download/package-manager/). Next we set up a directory for our project including a `contracts` sub-directory to hold our smart contract.   Then we initialize the npm package:

```sh
mkdir basil
cd basil
mkdir contracts
npm init --yes
```

## Truffle

We install Truffle, a crucial tool set for creating Solidity contracts and deploying them:

```sh
npm install -g truffle
```

## Truffle configuration file

Before we can do any contract deployments Truffle needs to know where to deploy our smart contracts.  Using your favorite editor, create a file named `truffle.js` and put the following content in it:

```
module.exports = {
  networks: {
    local: {
      host: "127.0.0.1",
      port: 8545,
      network_id: "*" // Match any network id
    }
  }
};
```

The `networks` section has entries for each destination network that we intend to use for smart contract deployments.  Currently we have a single entry in that section for a Ganache client running locally using the default port number `8545`.  Now Truffle knows where to find the Ganache client when we use `local` as the destination for a smart contract deployment.  

## Prerequisites

We install the `ZOS` command line interface globally with the following command:

```sh
npm install --global zos
```

Next we install the Open Zeppelin contracts library:

```sh
npm install openzeppelin-zos
```

Then we install the ZeppelinOS contract library:

```sh
npm install zos-lib
```

## The sample contract

Let's write the contract to control the light bulb.  Create a file named `Basil.sol` in the `contracts` sub-directory and put the code shown below in it:

```sol
/**
 * @title Basil
 */
 
pragma solidity ^0.4.21;
 
 
import "openzeppelin-zos/contracts/ownership/Ownable.sol";

contract Basil is Ownable {
    
    // color
    uint256 public r;
    uint256 public g;
    uint256 public b;
    
    // highest donation in wei
    uint256 public highestDonation;
    
    event Withdrawal(address indexed wallet, uint256 value);
    event NewDonation(address indexed donor, uint256 value, uint256 r, uint256 g, uint256 b);
    
    function donate(uint256 _r, uint256 _g, uint256 _b) public payable {
        require(_r < 256);
        require(_g < 256);
        require(_b < 256);
        
        r = _r;
        g = _g;
        b = _b;
        
        // Donation must be higher than the current highest
        require(msg.value > highestDonation);
        
        emit NewDonation(msg.sender, msg.value, r, g, b);
    }
    
    function withdraw(address wallet) public onlyOwner {
        require(address(this).balance > 0);
        require(wallet != address(0));
        uint256 value = address(this).balance;
        wallet.transfer(value);
        emit Withdrawal(wallet, value);
    }    
}
```

Save the file and make sure you're back in the `basil` sub-directory afterwards.  The contract is super simple.  If somebody wants to set the light color, they have to make a donation.  If the donation is higher than the previous one, it is accepted, the light color changes and an event is emitted.  Of course, the `withdraw` method allows the Zeppelin team to collect all donations, which are safely put away in the plant's own education fund.

We compile the contract we just created:

```sh
truffle compile
```

> NOTE: If you're familiar with the `openzeppelin-solidity` library, note that we're using something different here. We're using the `openzeppelin-zos` library, which is the OpenZeppelin version for ZeppelinOS.

## Using ZeppelinOS

 We initialize our application with the version 0.0.1 using the ZOS command line interface:

```sh
zos init basil 0.0.1
```

This will create a `zos.json` file where ZeppelinOS will keep track of
the contracts of your application.

Next, add the Basil contract to the list of contracts in `zos.json`:

```sh
zos add Basil
```

To have your `zos.json` file always up-to-date, run the `zos add` command for every
new contract you add to your project.  The json file should look like this now:

```json
{
  "name": "basil",
  "version": "0.0.1",
  "contracts": {
    "Basil": "Basil"
  }
}
```

OpenZeppelin will use this file to track your project's contracts on chain, making them upgradeable and dynamically linkable to pre-deployed libraries, as we'll see soon.

## Deploying our first version of Basil, locally

Let's start a local Ethereum network.  If you have never done this before, you will need to install the Ganache client using the following command:

```sh
npm install -g ganache-cli
```

Open a new Terminal window and execute the following command to start the Ganache client:

```sh
ganache-cli --deterministic
```

This will print 10 accounts. Copy the address of the first one and then go back to the terminal window that you used to create the Basil project.  Export the address you just copied to an environment variable named `OWNER` (see below).  You will need easy access to that address as you work through this tutorial:

```sh
export OWNER=<address>
```

Next, we deploy our app to Ganache:

```sh
zos push --from $OWNER --network local
```

The first time you run this command for a specific network, a new
`zos.<network>.json` will be created. This file will reflect the status
of your project in that specific network, including contract logic and instance addresses, etc.

## Contract logic and upgradeable instances

Notice how the file `zos.local.json` lists a series of "contracts" and "proxies". The `contracts` entry contains a list of each logic contract for each contract name.  The `address` field contains the deployment address of the logic contract and the `byteCodeHash` field contains the hash code for the contract.  Currently the `proxies` list is empty since we have not created any proxies yet.  A proxy is a wrapper for a logic contract.  By having a proxy, you can change or update a logic contract while maintaining its state.

We need to create an upgradeable instance (proxy) for Basil now.  We use the `zos create` command to create a proxy:

```
zos create Basil --from $OWNER --network local --init --args $OWNER
```

Copy the address this command outputs to the clipboard.  It should match the address in the line that is printed after the text `Basil proxy:`.  This is the address for the contract that is proxying our Basil contract.  We will need it later so export it to the environment with the following command:

```sh
export BASIL_PROXY_ADDRESS=0x9431db860bb727f7695ea8af6a4dc1d2cf34344d
```

We now have an entry in the `proxies` list for our Basil contract.  A proxy entry contains the deployment address for a specific logic contract, the current version number of that contract, and its implementation address.  Notice that the implementation address field in the proxy entry matches the deployment address field for the Basil entry in the `contracts list.  If contracts were mail, the implementation field could be considered the current "forwarding" address for the Basil logic contract.  When we upgrade the Basil contract later in this tutorial.  You can see this in the updated `zos.local.json` file, shown below:

```json
{
  "contracts": {
    "Basil": {
      "address": "0x1b88bdb8269a1ab1372459f5a4ec3663d6f5ccc4",
      "bytecodeHash": "089769261b0c6e1ab9eb20f3321f7cb959c4bbf9156bbfde92a31d2000a44689"
    }
  },
  "proxies": {
    "Basil": [
      {
        "address": "0x9431db860bb727f7695ea8af6a4dc1d2cf34344d",
        "version": "0.0.1",
        "implementation": "0x1b88bdb8269a1ab1372459f5a4ec3663d6f5ccc4"
      }
    ]
  },
  "app": {
    "address": "0xaf5c4c6c7920b4883bc6252e9d9b8fe27187cf68"
  },
  "version": "0.0.1",
  "package": {
    "address": "0x2d8be6bf0baa74e0a907016679cae9190e80dd0a"
  },
  "provider": {
    "address": "0x970e8f18ebfea0b08810f33a5a40438b9530fbcf"
  }
}
```

## Upgrading the contract

If we ever found a bug in Basil we would need to upgrade our zos package, provide a new implementation for Basil with a fix, and tell our proxy to upgrade to the new implementation. This would preserve all the previous donation history while seamlessly patching the bug.

Another common thing that happens when developing smart contracts for Ethereum is that new standards appear.  All the new kids implement them in their contracts and a very cool synergy between contracts starts to happen.  Developers who have already deployed immutable contracts will miss all the fun.  For example, it would be very nice to encourage donations to Basil by emitting a unique ERC721 token in exchange for the donation.  Let's upgrade the contract with ZeppelinOS to do just that.

We could modify `contracts/Basil.sol`, but now let's try something else.  We'll make a new contract name `BasilERC721.sol` in the `contracts` sub-directory that inherits from our initial version of Basil.  Create that file now and put the content shown below in it:

```sol
pragma solidity ^0.4.21;

import "./Basil.sol";
import "openzeppelin-zos/contracts/token/ERC721/MintableERC721Token.sol";
import "openzeppelin-zos/contracts/math/SafeMath.sol";

contract BasilERC721 is Basil {
  using SafeMath for uint256;

  // ERC721 non-fungible tokens to be emitted on donations.
  MintableERC721Token public token;
  uint256 public numEmittedTokens;

  function setToken(MintableERC721Token _token) external onlyOwner {
    require(_token != address(0));
    require(token == address(0));
    token = _token;
  }

  function donate(uint256 _r, uint256 _g, uint256 _b) public payable {
    super.donate(_r, _g, _b);
    emitUniqueToken(tx.origin);
  }

  function emitUniqueToken(address _tokenOwner) internal {
    token.mint(_tokenOwner, numEmittedTokens);
    numEmittedTokens = numEmittedTokens.add(1);
  }
}
```

A few things to note:

  * This new version extends from the previous one. This is a very handy pattern, because the proxy used in ZeppelinOS requires new versions to preserve the state variables.
  * We can add new state variables and new functions. The only thing that we can't do during a contract upgrade is to remove state variables.

Let's create a new version of our app, with the new contracts. Make sure you are back in the `basil` directory before executing this command.  This command simply changes the current version number for the app to "0.0.2" in the `zos.json` file:

```sh
zos bump 0.0.2
```

Let's add this version to our ZeppelinOS application and push to the network again.  Notice that we reference our base smart contract `Basil` when adding our new `BasilERC721` smart contract when using the `zos add` command. We do this by appending the base contract's name prefixed by a colon to the new contract's name as the primary argument tor the `zos add` command:

```sh
truffle compile
zos add BasilERC721:Basil
zos push --from $OWNER --network local
```
Now to upgrade our proxy and let it know about the new implementation:

```sh
zos upgrade Basil --from $OWNER --network local
```

Let's look a `zos.local.json` again.  Notice that the deployment address field for the Basil logic contract has changed to the new deployment address.  Because of this, the implementation field for the Basil proxy contract changed to match that new deployment address, thus preserving the linkage between the logic contract and the proxy contract.

From here on Basil's proxy will use the new implementation.  In its current state, Basil will revert on every donation because it doesn't have a token set for it yet.  We'll do that next.

## Connecting to OpenZeppelin's standard library

So far, we've used ZeppelinOS to seamlessly upgrade our app's contracts.  We will now use it to create a proxy for a pre-deployed ERC721 token implementation.

The first thing we need to do is to tell our app to link to the `openzeppelin-zos` standard library release.  Note, the link command may take a little while to complete:

```sh
zos link openzeppelin-zos
zos push --from $OWNER --deploy-stdlib --network local
```
If you look at the `zos.json` file now, you will see a new entry named `stdlib` that keeps track of the package name and current version number for the `openzeppelin-zos` package we just linked to:

```json{
  "name": "basil",
  "version": "0.0.2",
  "contracts": {
    "Basil": "BasilERC721"
  },
  "stdlib": {
    "name": "openzeppelin-zos",
    "version": "1.9.1"
  }
}
```

Notice the `--deploy-stdlib` option we used during the deployment using the `zos push` command.  This option injects a version of the ZeppelinOS standard library of contracts into our development network.  Since we're working on a local blockchain the smart contracts that make up the ZeppelinOS library don't exist there.  This handy option solves that problem for us quite nicely.

Now we create a proxy for the token:

```sh
zos create MintableERC721Token --from $OWNER --init --args \"$BASIL_PROXY_ADDRESS\",\"BasilToken\",\"BSL\" --network local
```

This command will output the token's new proxy address.  It should match the address from the line that begins with the label `MintableERC721Token proxy`.   The command below conveniently fills in the needed values from the environment variables we set earlier, and then passes that command to a new Truffle console session, to be executed immediately:

```sh
export TOKEN_PROXY_ADDRESS=<address>
echo "BasilERC721.at(\"$BASIL_PROXY_ADDRESS\").setToken(\"$TOKEN_PROXY_ADDRESS\", {from: \"$OWNER\"})" | truffle console --network local
```
You will see the result of the transaction in JSON format with some interesting details like how much gas was used, the block number, and other information.


That's it! Now you know how to use ZeppelinOS to develop upgradeable apps. Have a look at the scripts `deploy/deploy_with_cli_v1.sh` and `deploy/deploy_with_cli_v2.sh` to review what we've gone over in the guide.

Stay tuned for more advanced tutorials!
