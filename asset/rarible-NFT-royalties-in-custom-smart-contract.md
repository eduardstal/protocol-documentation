
A goal for us as creators is to create pieces that have continued value in the eyes of our buyers. This goes beyond the primary sale and continues into all subsequent secondary sales.

Smart contracts, which are the underlying foundation to all NFTs, allow us to integrate many amazing features into our work, and luckily programmable royalties is one of them! This allows us to benefit from each time a new buyer purchases our existing NFT.

This tutorial is meant to guide you in creating your own NFT custom smart contract that includes a built in functionality to always collect royalties when it is sold on the Rarible marketplace. Let’s get started!

### Prerequisite and set up

We will mainly be using the following tools in this tutorial:

* Node.js — Javascript engine (for running our JS programs)
* Hardhat — smart contract developer environment
* OpenZeppelin — audited smart contract templates

First step is to install Node.js (≥14 version) if you do not have it already. You can use this [link](https://nodejs.org/en/) to find the installer.

Now in an empty directory initiate an NPM project with 

```diff
npm init -y
```
Once the project is set up, install Hardhat with:
```diff
npm install --save-dev hardhat
```

After Hardhat is installed follow up with:
```diff
npx hardhat
``` 
And select the option to *Create a basic sample project* . This will include the necessary file structure for us to easily create, test, and deploy our own contracts.

![image](https://user-images.githubusercontent.com/39627934/132372512-2bbf5deb-9bc8-4b79-aca5-b2725f06f545.png)

Select Create a basic sample project

You can delete the file `Greeter.sol` found in the contracts folder and remove the file `sample-test.js` from the test folder.

We are also going to install two more libraries that are [Hardhat plugins](https://hardhat.org/plugins/). They allow us to add utilities for testing and deploying our smart contract.

```diff
npm install --save-dev @nomiclabs/hardhat-waffle @nomiclabs/hardhat-ethers
```

At the top of your `hardhat.config.js` file, add

```diff
require(“@nomiclabs/hardhat-waffle”); //this should already be there
```
```diff
require(“@nomiclabs/hardhat-ethers”);
```

We will also install a package called [chai](https://www.chaijs.com/), which will be used later for testing our smart contract.

```diff
npm install --save-dev chai
```

Let’s make some more adjustments to the `hardhat.config.js` file that will help us later on. These updates will specify the the version of Solidity and the settings we will compile with. Replace the existing module.exports with the following:

```diff
module.exports = {
    solidity: {
        version: "0.8.4",
    },
    settings: {
       optimizer: {
           enabled: true,
           runs: 999999,
        },
    },
};
```

# Creating the ERC721 smart contract

Let’s install the OpenZeppelin smart contract library.

```diff
npm install --save-dev @openzeppelin/contracts
```

This will give us access to all of the current OpenZeppelin contracts, which have been audited for security and I recommend as a starting point for pretty much any smart contract.

![image](https://user-images.githubusercontent.com/39627934/132373382-ab56cadd-86a3-4d95-9b1d-3aa62cb2fde0.png)

Now create a file in the contracts folder and name it `RaribleRoyaltyERC721.sol` .

## `RaribleRoyaltyERC721.sol`

In the file, let’s create a simple ERC721 that imports the ERC721 template provided by OpenZeppelin.

```diff
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
contract RoyaltyNFT is ERC721 {
    constructor() ERC721("RoyaltyNFT", "ROY") {}
}
```

We also want to add a few more useful functionalities to our smart contract.

The first will make it Ownable, so that only the smart contract owner can call certain functions. Add the following line just below our first import statement:

```diff
import “@openzeppelin/contracts/access/Ownable.sol”;
```

The second will add an internal counter, so that each time you mint a new token, the Token ID will increment by 1. Add the following line just below our second import statement:

```diff
import “@openzeppelin/contracts/utils/Counters.sol”;
```

Now that we have imported these OpenZeppelin utilities, let’s update the contract to be Ownable, properly increment on each token mint, have a function to mint new tokens, and have a function to set the baseURI.

*The baseURI won’t actually be used in this tutorial, but it is always good practice to set it, hence why I included it here.*

```diff
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
contract RoyaltyNFT is ERC721, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIdTracker;
    
    constructor() ERC721("RoyaltyNFT", "ROY") {}
    function mint(address _to) public onlyOwner {
        super._mint(_to, _tokenIdTracker.current());
        _tokenIdTracker.increment();        
    }
    function _baseURI() internal view virtual override returns (string memory) {
        return "https://exampledomain/metadata/";
    }
}
```

Now let’s make sure it can compile!

In the scripts folder there should be a file called sample-script. We didn’t deleted this earlier because it contains a very useful pattern to reuse.

First rename the file to `RaribleRoyaltyERC721-script.js` . Then within the file remove everything within the function called main(), and replace with the following code:

```diff
async function main() {
    const RoyaltyNFT = await hre.ethers.getContractFactory("RoyaltyNFT");
    const royaltyNFT = await RoyaltyNFT.deploy();
    await royaltyNFT.deployed();
    console.log("RoyaltyNFT deployed to:", royaltyNFT.address);
}
```

Your script file should look like this below image.

![image](https://user-images.githubusercontent.com/39627934/132373961-0a744852-2022-4fad-93af-779e6a282896.png)

Check that the contract compiles successfully with the terminal command 
```diff
npx hardhat compile
``` 
This will create two additional folders in your folder structure named *artifacts* and *cache* .

Time to add royalties!

# Integrating Rarible royalties

Rarible manages royalties on-chain (i.e., via smart contracts) through its own royalty registry contract. Every time a transfer occurs within the marketplace, the Rarible protocol does a check to see if the token expects to have any royalties paid out.

These contracts are all available at https://github.com/rarible/protocol-contracts/tree/master/royalties. An NPM package for Rarible Royalties does exist, but it is currently written for versions of Solidity below 0.8.0, so in our case we will make our own!

In your contracts folder, create a new folder structure *@rarible/royalties/contracts* . Then add the following files:

## `IRoyaltiesProvider.sol`

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./LibPart.sol";interface IRoyaltiesProvider {
    function getRoyalties(address token, uint tokenId) external         returns (LibPart.Part[] memory);
}
```

## `LibPart.sol`

```diff
// SPDX-License-Identifier: MIT 
pragma solidity ^0.8.0;
library LibPart { 
    bytes32 public constant TYPE_HASH = keccak256(“Part(address account,uint96 value)”);
    struct Part { address payable account;
        uint96 value; 
    }
    function hash(Part memory part) internal pure returns (bytes32){ 
        return keccak256(abi.encode(TYPE_HASH, part.account,  part.value));
    }
}
```

## `LibRoyaltiesV2.sol`

```diff
// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.0;
library LibRoyaltiesV2 {
    /*
    * bytes4(keccak256('getRoyalties(LibAsset.AssetType)')) == 0x44c74bcc
    */
    bytes4 constant _INTERFACE_ID_ROYALTIES = 0x44c74bcc;
}
```

## `RoyaltiesV2.sol`

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;import "./LibPart.sol";  
interface RoyaltiesV2 {
    event RoyaltiesSet(uint256 tokenId, LibPart.Part[] royalties);
    function getRaribleV2Royalties(uint256 id) external view returns (LibPart.Part[] memory);
}
```

Then add another folder called “impl” and add the following files in there:

## `impl/AbstractRoyalties.sol`

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "../LibPart.sol";
abstract contract AbstractRoyalties {
    mapping (uint256 => LibPart.Part[]) public royalties;
    function _saveRoyalties(uint256 _id, LibPart.Part[] memory _royalties) internal {
        for (uint i = 0; i < _royalties.length; i++) {
            require(_royalties[i].account != address(0x0), "Recipient should be present");
            require(_royalties[i].value != 0, "Royalty value should be positive");
            royalties[_id].push(_royalties[i]);
        }
        _onRoyaltiesSet(_id, _royalties);
    }
    function _updateAccount(uint256 _id, address _from, address _to) internal {
        uint length = royalties[_id].length;
        for(uint i = 0; i < length; i++) {
            if (royalties[_id][i].account == _from) {
                royalties[_id][i].account = payable(address(uint160(_to)));
            }
        }
    }
    function _onRoyaltiesSet(uint256 _id, LibPart.Part[] memory _royalties) virtual internal;
}
```

## `impl/RoyaltiesV2Impl.sol`

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./AbstractRoyalties.sol";
import "../RoyaltiesV2.sol";
contract RoyaltiesV2Impl is AbstractRoyalties, RoyaltiesV2 {
    function getRaribleV2Royalties(uint256 id) override external view returns (LibPart.Part[] memory) {
        return royalties[id];
     }
    function _onRoyaltiesSet(uint256 _id, LibPart.Part[] memory _royalties) override internal {
        emit RoyaltiesSet(_id, _royalties);
     }
}
```

If you have added everything, then your folder structure should look like this:

![image](https://user-images.githubusercontent.com/39627934/132374441-1beb4b81-c089-4041-bdcf-0212ea7c74a0.png)

It aligns to the same structure as Rarible’s github repo, though our contracts have been upgraded to Solidity ≥0.8.0!

As mentioned above, during a NFT sale the Rarible Protocol will determine if the token expects royalties paid out. This is managed by the [RoyaltiesRegistry ](https://github.com/rarible/protocol-contracts/blob/master/royalties-registry/contracts/RoyaltiesRegistry.sol)smart contract, which checks if the token’s smart contract contains the appropriate interface and calls a function named `getRaribleV2Royalties` .

So in our own smart contract let’s import the appropriate Rarible contracts, add a function to set a royalty per token, and add a function to allow Rarible’s RoyaltyRegistry contract to call and receive the royalty amount.

```diff
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "./@rarible/royalties/contracts/impl/RoyaltiesV2Impl.sol";
import "./@rarible/royalties/contracts/LibPart.sol";
import "./@rarible/royalties/contracts/LibRoyaltiesV2.sol";
contract RoyaltyNFT is ERC721, Ownable, RoyaltiesV2Impl {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIdTracker;
    
    constructor() ERC721("RoyaltyNFT", "ROY") {}
    function mint(address _to) public onlyOwner {
        super._mint(_to, _tokenIdTracker.current());
        _tokenIdTracker.increment();        
    }
    function _baseURI() internal view virtual override returns (string memory) {
        return "https://exampledomain/metadata/";
    }
    function setRoyalties(uint _tokenId, address payable _royaltiesReceipientAddress, uint96 _percentageBasisPoints) public onlyOwner {
        LibPart.Part[] memory _royalties = new LibPart.Part[](1);
        _royalties[0].value = _percentageBasisPoints;
        _royalties[0].account = _royaltiesReceipientAddress;
        _saveRoyalties(_tokenId, _royalties);
    }
    function supportsInterface(bytes4 interfaceId) public view virtual override(ERC721) returns (bool) {
        if(interfaceId == LibRoyaltiesV2._INTERFACE_ID_ROYALTIES) {
            return true;
        }
        return super.supportsInterface(interfaceId);
    }
}
```


In our function setRoyalties, the parameter `_percentageBasisPoints` sets the percentage value of the royalty — e.g., an input of 1000 equals a royalty of 10%.

Now that we have the royalty functionality integrated into our smart contract, let’s test that it works!

# Testing the smart contract

In the world of blockchain, testing is essential as you are often dealing with other people’s assets. It is your responsibility as the smart contract developer to make sure everything works correctly to the best of your knowledge. Here we will conduct a simple test to ensure the royalty functionality works as it is intended.

In the test folder, create a file named `RaribleRoyaltyERC721-test.js` . We will create a simple test that will deploy the smart contract, mint two tokens, and set a different royalty to each token. In your newly created file paste the following code:

```diff
const { expect } = require('chai')
const { ethers } = require('hardhat')
describe('RaribleRoyaltyERC721 Test', () => {
    let royaltyNFT
    before(async () => {
        [deployer, account1, account2] = await ethers.getSigners()
        const RoyaltyNFT = await ethers.getContractFactory("RoyaltyNFT")
        royaltyNFT = await RoyaltyNFT.deploy()
await royaltyNFT.deployed()
})
    describe('Mint token and set royalty', async () => {
        it('mint two tokens and set two different royalties', async () => {
            const royalty10Percent = 1000
            const royalty20Percent = 1000
            await royaltyNFT.connect(deployer).mint(account1.address)
            await royaltyNFT.connect(deployer).setRoyalties(0, account1.address, royalty10Percent)
            await royaltyNFT.connect(deployer).mint(account1.address)
            await royaltyNFT.connect(deployer).setRoyalties(1, account1.address, royalty20Percent)
            const token0Royalty = await royaltyNFT.getRaribleV2Royalties(0)
            const token1Royalty = await royaltyNFT.getRaribleV2Royalties(1)
            expect(token0Royalty[0][1]).to.equal(royalty10Percent)
            expect(token1Royalty[0][1]).to.equal(royalty20Percent)
        })
    })
})
```

In your terminal run the test with the following command: 
```diff
npx hardhat test
```

If the contracts have been set up correctly the you should get a passing test.

![image](https://user-images.githubusercontent.com/39627934/132374730-8a1eb90f-3579-4f4a-a79d-2a26e0970060.png)

In your own project, you would want to do several more tests depending on the complexity of your contract, including deploying on test nets (e.g., Ropsten). Luckily Rarible is available on most test nets (e.g., https://ropsten.rarible.com/), so you can deploy your NFT on a test net and use the Rarible Marketplace UI to make sure everything works properly.

# Final words

To finalize your ERC721, you would want to associate your tokens with some type of metadata by ensuring each token’s URI points to a metadata json file. You would also want to further update your hardhat.config.js file and script file(s) to deploy your contract on other networks outside of the Hardhat environment.

If you want to see a smart contract that has this royalty functionality built in, then check out the NFT project [Aisthisi](https://aisthisi.art/).

The project that we created today is also available on [github](https://github.com/AboldUSER/rarible-royalties-tutorial). If you found this tutorial helpful then be sure to give this article a few claps and a star on the github repo. ## To be edited ++ add credits 
