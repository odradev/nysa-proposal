# Nysa Proposal
Nysa aims to transpile smart contracts written in Solidity into Rust code.
Thanks to this approach the output smart contract will be as efficient as
a contract written using pure [near-sdk-rs](https://github.com/near/near-sdk-rs/).
Our first experiments showed promising results.
We have the example, that is 25 times more gas efficient than Aurora.

## What problem do we solve?
The main advantage of smart contracts written in Solidity is that they are already
written and battle-tested - hence considered secure.
Platforms that offer writing smart contracts in Rust suffer from small amount
of libraries and example code, so they look at Solidity codebase with jealousy.
To take advantage of the existing codebase, Aurora approach translates EVM bytecode
into WASM bytecode. The main issue of this solutions is the efficiency. 

## What do we offer?
We strongly believe there is a better solution: **transpiling Solidity code into Rust** code.
It allows to take advantage of the already existing Rust to WASM compilation pipeline.
In return, we get the same neat, easy and secure but native code.

## How we do it?
In terms of programming langauges the list of Solidity features can be treated
as a subset of Rust's features.
That means it's possible to convert Solidity into Rust, but the oposite is not possible.
Our architecture is split into 3 main pieces:

- `Solidity Parser` build on top of [lalrpop](https://github.com/lalrpop/lalrpop).
It parses Solidity Syntax into developer friendly Solidity AST.
We have borrowed it as a whole from [the Solang project](https://github.com/hyperledger/solang).
Further development might require some modification, but it can be considered 95% complete.

- `C3 Lang` is our approach to implement Solidity's (and Python's) inheritence model.
We designed it as a intermediete target for the transpiler.
While it works, it needs better tests, redesign and documentation.

- `Nysa Core` translates the output of the `Solidity Parser` into a `C3 Lang` Rust.

The pipeline looks like this:
```
Solidity Code -> Solidity Parser -> Solidity AST -> C3 Lang AST -> Rust Code
```

## Nysa codebase
We managed to implement the bare minimum to transpile a simple Solidity code.
It lives in [the Nysa repository](https://github.com/odradev/nysa).

## Fibonacci example
We have a non-trivial example implemented - fibonacci sequence.

Solidity code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Fibonacci {
    mapping(uint32 => uint32) results;

    function compute(uint32 input) public payable {
        results[input] = fib(input);
    }

    function get_result(uint32 input) public view returns (uint32) {
        return results[input];
    }

    function fib(uint32 n) public returns (uint32) {
        if (n <= 1) {
            return n;
        } else {
            return fib(n - 1) + fib(n - 2);
        }
    }
}
```

It corresponds to the following Rust code:
```rust
#[near_bindgen]
#[derive(Default, BorshDeserialize, BorshSerialize)]
pub struct Fibonacci {
    results: HashMap<u32, u32>,
}

#[near_bindgen]
impl Fibonacci {
    pub fn compute(&mut self, input: u32) {
        self.results.insert(input, self.fibb(input));
    }

    pub fn get_result(&self, input: u32) -> u32 {
        self.results.get(&input).cloned().unwrap_or_default()
    }

    fn fibb(&self, n: u32) -> u32 {
        if n <= 1 {
            return n;
        } else {
            return self.fibb(n - 1) + self.fibb(n - 2);
        }
    }
}
```

Full code with tests lives [here](https://github.com/odradev/nysa/tree/master/example-fibonacci/src).

### Auroria vs Nysa
We have compared the execution of `compute(18)` functions on the NEAR Testnet.

Running it via Aurora used `249 Tgas` and cost `0.02488 yocto`.
[Aurora TX](https://testnet.aurorascan.dev/tx/0xad61caede25e414011db9466231a10bf28edaf77bec777e00fd64c1fafa3beff),
[Testnet TX](https://explorer.testnet.near.org/transactions/2gprMYnkLvxDTbycqGr7j4BBeu4YT4dm9LE6TTj5Qd2y).

Running transpiled code on Testnet directly used `9 Tgas` and cost `0.00096 yocto`.
[Testnet TX](https://explorer.testnet.near.org/transactions/6L7NGVqU2WT9kvYW6WPsjpP3zdNhe61WaEiPsEUdHAY9).

# Timeline
Proposed list of milestones.

## Milestone #1 MVP (Flipper)
Basic version of Solidity Flipper contract, which is compatible with Nysa.
The code can be transpiled to Rust and compiled into a wasm file.
In compare to the current code it will be redesigned and prepared for
the large-scale development.

Covered Solidity features:
- Functions (modifiers and constructors, Function Calls, returns).
- Contract state variables (nested mappings).
- Visibility and getters of state variables.
- Basic types.

## Milestone #2 MVP 2.0 (Flipper 2.0)
Advanced version of Solidity Flipper contract, which uses local variables,
error handling, control structures and operators is compatible with Nysa.

Covered Solidity features:
- Checked/unchecked arithmetic.
- Local variables.
- Error handling.
- Control Structures.
- Operators.

## Milestone #3 Inherited Flipper
Solidity Flipper contract now implements an Interface and inherits an abstract Flipper contract.
Additionally, Pragma section is read by transpiler.
Contracts written in Solidity below 0.8.0 are rejected, >=0.8.* are compiled.

Covered Solidity features:
- Interfaces.
- Inheritance.
- Abstract Contracts.
- Pragma.

## Milestone #4 ERC20
ERC20 from OpenZeppelin contract is compatible with Nysa.
This milestone requires working events.
Additionally, we wish to implement support for imports and external contract calls.

Covered Solidity features:
- Imports.
- Events.
- External contract calls.

## Milestone #5 Structs and scoping
This milestone focuses on more advanced solidity features, such as Special Variables and Functions,
conversions between elementary types.
At this point conversion of structs and enums will be available.
Solidity contracts which uses those features are compatible with Nysa.

## Milestone #6 Function modifiers
This milestone focuses on some unique Solidity features like function modifiers,
time units, ether units, and immutable state variables.

## Milestone #7 Libraries
This milestone aims to cover the missing solidity features like libraries, `using for` keyword,
and other advanced langauge structures. Secondary goal is to fully cover with tests
the order of evaluation of expressions.

## Milestone #8 OpenZeppelin
Solidity contracts written using OpenZeppelin framework are compatible with Nysa.

## Milestone #8 Docs
The documentation of all the supported features and discrepancies to EVM are documented.
It includes tutorials and introduction for Solidity developers. 

## Milestone #9 Solidity tests
Tests written in solidity can be transpiled into Rust tests.

## Milestone #10 JS E2E tests
E2E JavaScript tests written for OpenZeppelin can be run against the NEAR Virtual Machine.
The tests are passing.
