# Building an Upgradeable Contract

---

## Introduction
The concept of Smart Contract being immutable is a great idea that create enough trust to users interacting with a deployed implementation. This concept has its own downside with creator/developer not having the capability to fix any vulnerability when the case arises (since developer can not change anything on a deployed contract), giving attacker an unfair advantage over the developer. Upgradeable Contract provides solution to this smart contract problem while giving developers a good experience to maintain the immutable nature of smart contracts and logic of their implementation.

## Table of Contents
- [Building an Upgradeable Contract](#building-an-upgradeable-contract)
  - [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [Objective](#objective)
  - [Prerequisites](#prerequisites)
  - [Requirements](#requirements)
  - [Tutorial](#tutorial)
    - [STEP 1 - Spin off Hardhat Environment](#step-1---spin-off-hardhat-environment)
    - [STEP 2 - Setup Upgradeable Contract](#step-2---setup-upgradeable-contract)
    - [STEP 3 - Simple bank contract with Withdraw defect](#step-3---simple-bank-contract-with-withdraw-defect)
    - [STEP 4 - Interacting with our deployed Contract](#step-4---interacting-with-our-deployed-contract)
    - [STEP 5 - Upgrade Celo Bank contract to solve bug](#step-5---upgrade-celo-bank-contract-to-solve-bug)
    - [Conclusion](#conclusion)
## Objective
By the end of this tutorial you should be able to write an upgradeable smart contract using the **Universal Upgradeable Proxy Standard (UUPS)**

## Prerequisites
* Knowledge of using Hardhat is important
* Intermediate/advance knowledge in Solidity
* Basic knowledge of using the command line
* Understanding delegateCall
* Understanding of vscode is a must

## Requirements
* Have Node.js installed from version V10. or higher
* Have npm or yarn installed
* vscode

## Tutorial
### STEP 1 - Spin off Hardhat Environment
The first thing we will be doing is to create a folder for our implementation and spin off hardhat environment, go to your terminal and follow this processes below to create an environment for our implementation

```
mkdir uupsPractice
```

This command create a new folder for us, we then need to initialize and open the folder either through the terminal using the command
```
 npm init -y
 cd uupsPractice
 code .
```

or from our vscode.
N.B: I'm using a linux base operating system but this should also work for windows (They may be a slight differences)
![](https://i.imgur.com/c5L69ZQ.png)

After opening the folder in our vscode, we should have something similar to what we have below.
![](https://i.imgur.com/UVWJOm6.png)

Now let's complete our initialization process by spinning off our hardhat environment, I will be using npm.
```
 npm install hardhat
 npx hardhat
```
after the command `npx hardhat` we should see something like what we have below.
![](https://i.imgur.com/o3XT3Pi.png)

For this tutorial we will be using "create a typescript project", confirm this and other prompt options.

### STEP 2 - Setup Upgradeable Contract

We will be using a simple bank contract for this tutorial as it will give a better understanding on implementing the upgradeable standard.

[Click here](https://eips.ethereum.org/EIPS/eip-1822) to know more about UUPS.

Within our contract folder, we will create two files named
1. proxy.sol (The proxy contract that delegate call to our implementation contract)
2. proxiable.sol (Responsible for upgrading contract)

**What is Proxy contract**
Proxy contract is a contract (Contract A) that delegates call to another contract (Contract B) while maintaining the same storage layout(state variables).
![](https://i.imgur.com/pcKYn2W.png)

in our proxy.sol contract, copy and paste the code below
```
//SPDX-License-Identifier: MIT

pragma solidity 0.8.18;

contract Proxy {
    // Code position in storage is keccak256("PROXIABLE") = "0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7"

    constructor(address contractLogic) {
        // save the code address
        assembly { // solium-disable-line
            sstore(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7, contractLogic)
        }
    }

    fallback() external payable {
        assembly {
            let contractLogic := sload(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7)
            calldatacopy(0x0, 0x0, calldatasize())
            let success := delegatecall(sub(gas(), 10000), contractLogic, 0x0, calldatasize(), 0, 0)
            let retSz := returndatasize()
            returndatacopy(0, 0, retSz)
            switch success
            case 0 {
                revert(0, retSz)
            }
            default {
                return(0, retSz)
            }
        }
    }

    receive() external payable {}
}
```
![](https://i.imgur.com/Ho4OTfu.png)

Before explaining this code it's important to have some understanding on delegate call, [this article gives a better explanation on delegate call](https://medium.com/coinmonks/delegatecall-calling-another-contract-function-in-solidity-b579f804178c).

The constrcutor has two parameters, *constructData* and *contractLogic*. The *constructData* takes in  the initialization calldata/payload (representing a constructor in our implementation), the *contractLogic* takes in implemetation contract address (address to delegatecall to).

**A fallback function** triggered when a non-existence function is called in a contract. Whenever a user interact with proxy contract by calling a function that does not exist in the contract the fallback function will be triggered, and the logic within the fallback is being computed, giving us the ability to delegatecall to our implementation contract 

in our *proxiable.sol*, copy and paste the code below
```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.18;


contract Proxiable {
    // Code position in storage is keccak256("PROXIABLE") = "0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7"

    function updateCodeAddress(address newAddress) internal {
        require(
            bytes32(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7) == Proxiable(newAddress).proxiableUUID(),
            "Not compatible"
        );
        assembly {
            sstore(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7, newAddress)
        }
    }
    function proxiableUUID() public pure returns (bytes32) {
        return 0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7;
    }
}
```
*proxiable.sol contract* will be inherited by our logic contract and it is responsible for upgradeability

> - The `updateCodeAddress` function updates a contract's implementation address by storing the new address at the location corresponding to the keccak256 hash of the string "PROXIABLE" in storage. The function checks if the new implementation contract is compatible with the Proxiable contract by comparing its UUID with the expected value, and then sets the new address using the sstore opcode.
>   
> - The `proxiableUUID` function returns the expected UUID for the implementation contract, which is the same keccak256 hash as the one used to store the code position in storage.

### STEP 3 - Simple bank contract with Withdraw defect
We will create our simple bank contract file using the name *celoBank.sol*, then copy and paste what we have below.
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;


import {Proxiable} from "../contracts/proxiable.sol";

contract celoBank is Proxiable{

    mapping(address => uint256) usersDeposit;

    address owner;
    bool initialized;
    

    function init(address _addr) external {
        require(!initialized, "Already initialized");
        owner = _addr;
        initialized = true;
    }

    function admin() external view returns(address) {
        return owner;
    }


    function deposit() external payable {
        require(msg.value != 0, "Amount can't be zero");
        usersDeposit[msg.sender] += msg.value;
    }

    function userBalance(address _addr) external view returns(uint256){
        return usersDeposit[_addr];
    }

    function upgradeContract(address _newAddress) external {
        require(msg.sender == owner, "You are not an admin");
        updateCodeAddress(_newAddress);
    }
}
```

> The code defines a smart contract called celoBank, which inherits from Proxiable contract. The contract has a mapping named `usersDeposit`, which maps a user's address to their deposit. The contract also has an `owner` state variable that tracks owners address and an `initialized` boolean state variable.
> 
> The `init` function is used to set the owner address and is called only once during initialization. The `admin` function is used to return the owner address.
> 
> The `deposit` function is used to accept deposits from users and the deposited amount is added to the `usersDeposit` mapping for the respective user's address.
> 
> The `userBalance` function is used to get the deposited balance of a user's address.
> 
> Finally, the `upgradeContract` function is used to upgrade the contract implementation by calling the updateCodeAddress function from the Proxiable contract. Only the contract owner can call this function.
> 

Next is to setup our hardhat config. We need to install dotenv by pasting the code below in our terminal

> npm install dotenv

we can then create a file named *.env* and paste our private key in our *.env*. It should look like what we have below

![](https://i.imgur.com/oyfaIXR.png)

Now, to the main hardhat setup for our celo alfajores testnet, copy and paste the code below in hardhat.config.ts

```
import "@nomicfoundation/hardhat-toolbox";
require("dotenv").config({path: ".env"});


const PRIVATE_KEY = process.env.PRIVATE_KEY



module.exports = {
  solidity: "0.8.18",
  networks: {
    alfajores: {
      url: "https://alfajores-forno.celo-testnet.org",
      accounts: [PRIVATE_KEY]
    }
  }
}

```
Before moving to writing a script for our contract, we need to compile the code with this command in our terminal
> npx hardhat compile

let's go on to deploy and interact. In our *deploy.ts* copy and paste this code to deploy our contract on celo Alfajores testnet.

```
import { ethers} from "hardhat";

async function main() {

  const ABI = [
    "function init(address _addr)"
  ];

  const iProxy = new ethers.utils.Interface(ABI);

  const constructData = iProxy.encodeFunctionData("init", ["0x5DE9d9C1dC9b407a9873E2F428c54b74c325b82b"]);

  console.log(constructData, "construct data")

  // deploy implementation/logic contract
  const CeloBank = await ethers.getContractFactory("celoBank")
  const celoBank = await CeloBank.deploy()

  await celoBank.deployed()
  console.log(`Celo bank deployed to ${celoBank.address}`)


  // deploy proxy contract
  const Proxy = await ethers.getContractFactory("Proxy")
  const proxy = await Proxy.deploy(constructData,celoBank.address)

  await proxy.deployed()
  console.log(`Proxy contract deployed to ${celoBank.address}`);
  

}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});

```
> This script uses the Hardhat framework and Ethers.js library to deploy two contracts: celoBank and a proxy contract.
> - The `ABI` variable stores an array of function signatures of the celoBank contract.
> -  The `iProxy` variable uses the Interface method from Ethers.js to encode the init function signature of celoBank along with the constructor arguments into constructData.
> 
> Next, the CeloBank factory is obtained, and celoBank is deployed. The Proxy contract factory is also obtained, and a new instance of Proxy is deployed with the constructData and celoBank's address as arguments. Finally, the script logs the deployment addresses of both contracts.

Since we have our script fully written, we need to paste this command below in our terminal
> npx hardhat run scripts/deploy.ts --network alfajores

**Note:** You must have test celo in your metamask wallet, before this operation can be succesful. You can get a test celo [here](https://faucet.celo.org/).

You should have something like what I have below

![](https://i.imgur.com/Pt9Wp70.png)


### STEP 4 - Interacting with our deployed Contract
We need to checkup our deployed contract on celo testnet explorer [here](https://alfajores.celoscan.io/) to confirm our deployment.
![](https://i.imgur.com/RACLxV5.png)

After deployment has been confirmed, it is necessary for us to verify, so we can interact on the explorer and make our deployment readable to others.

To verify, click on the contract tab and open the link to *verify and publish* as shown below

![](https://i.imgur.com/N7yLCvP.png)

Follow the procedures to verify (It's straight forward by following the guildlines). Verify both the implementation/logic contract (celoBank contract) and the proxy contract deployment.

When you are done with the proxy contract verification, you should see something like this below.
![](https://i.imgur.com/MBzUcM6.png)

select more options and make the explorer see it as a proxy contract. At the end it should look like what we have below.

![](https://i.imgur.com/6EMTQan.png)

One thing you will notice is that we now have all functions available on our implementation contract here, even thought those functions are not written in our *proxy.sol*. changing the implementation contract address to a new contract address will change the available functions in proxy with a persistent storage.

Let's interact with our deployed proxy by depositing some celo and view the user balance after.
- we connect our explorer to our wallet
- we deposit some celo to the celoBank contract
![](https://i.imgur.com/M8XI0oj.png)
- Lastly, we view user's balance
![](https://i.imgur.com/Omxsrer.png)

### STEP 5 - Upgrade Celo Bank contract to solve bug

While interacting we noticed users can deposit but cannot withdraw their funds. Traditionally, the money will be locked in the contract forever, but since this is an upgradeable contract, we can upgrade our contract to implement withdrawal.

This step, will guide us through on how to upgrade our contract.

NOTE: The upgrade can only be done by the admin. Without the admin, the upgrade is not possible.

let's go on to create another implementation with withdrawal option. name it celoBankUpgrade.sol and paste the code below
```
// SPDX-License-Identifier: MIT


pragma solidity 0.8.18;
import {Proxiable} from "../contracts/proxiable.sol";

contract celoBankUpgrade is Proxiable{

    mapping(address => uint256) usersDeposit;

    address owner;
    bool initialized;
    

    function init(address _addr) external {
        require(!initialized, "Already initialized");
        owner = _addr;
        initialized = true;
    }

    function admin() external view returns(address) {
        return owner;
    }


    function deposit() external payable {
        require(msg.value != 0, "Amount can't be zero");
        usersDeposit[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 myBalance = usersDeposit[msg.sender];
        usersDeposit[msg.sender] = 0;
        (bool success, ) = payable(msg.sender).call{value: myBalance}("");
        require(success, "Invalid");
    }

    function userBalance(address _addr) external view returns(uint256){
        return usersDeposit[_addr];
    }

    function upgradeContract(address _newAddress) external {
        require(msg.sender == owner, "You are not an admin");
        updateCodeAddress(_newAddress);
    }
}
```

now that we have created a new contract with withdraw function, we need to deploy our *celoBankUpgrade.sol*. Create a deploy file with a name *deployUpgrade.ts* and paste the following code below

```
import { ethers} from "hardhat";


async function main() {
    const CeloBankUpgrade = await ethers.getContractFactory("celoBankUpgrade");
    const celoBankUpgrade = await CeloBankUpgrade.deploy();

    await celoBankUpgrade.deployed();

    console.log(`celo bank upgrade deployed to ${celoBankUpgrade.address}`)
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
If everything is right, should look like what we have below
![](https://i.imgur.com/GrnT2Lp.png)

paste the deployment command below in your terminal
```
npx hardhat run scripts/deployUpgrade.ts --network alfajores
```
We can go on to verify as we did for celoBank contract and proxy contract.

Now, let's upgrade our proxy contract by passing celoBankUpgrade contract address as a parameter to upgradeContract function in proxy contract.

![](https://i.imgur.com/kLANPxI.png)

After upgrading, you can see that we now have our withdraw function available which will give us the opportunity to withdraw our deposited funds (Remeber, we already deposited 1 celo before the upgrade).

Now we can withdraw it through the power of upgradeable contract.

![](https://i.imgur.com/syOwpEc.png)

### Conclusion
Upgradeable contract gives us the flexibility to fix bugs in an immutable contract as we see in our practice. It is necessary to take this practice beyond creating simple bank contract and use it for real life solution.

UUPS is not the only upgradeable contract standard. We have
- Diamond Standard (little bit advanced than UUPS)
- Transparent Upgradeable Proxy


Thank you for following up till the end of this tutorial, I hope you learn something new.

Here's a link to the project [UUPS for celo blockchain](https://)


