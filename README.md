# To-do list Æpp

## Overview

This is a simple To-do list Aepp that demonstrates how to interact with already deployed Sophia smart contract.

## Installation

1. Clone this repo and install required dependencies
```bash
git clone https://github.com/aeternity/to-do-list/
cd to-do-list-aepp
yarn install
```
1. Add a deployed contract address, byte code and a host to `src/settings.js`
```
export default {
  host: 'https://sdk-testnet.aepps.com',
  deployedContractAddress: CONTRACT_ADDRESS,
  deployedContractByteCode: CONTRACT_BYTE_CODE
}
```

## Start the application

```
yarn run start:dev
```

## Aepp  functionalities

- connect to host provided in the ```settings.js``` file;
- check the balance of the account;
- create a to-do task;
- mark a to-do task as completed;

