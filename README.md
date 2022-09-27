# Nysa Proposal
Nysa is

## Universal design.

## What problem do we want to solve?

## What do we want to offer?
To win developers' hearts, we want to offer:
1. One
2. Two

# Nysa


# Timeline
Proposed list of milestones.

## Milestone #1 MVP (Flipper)
Basic version of Solidity Flipper contract, which is compatible with Nysa, which means that the code can be transpiled to rust, compiled into a wasm file and deployed into NEAR testnet.

## Milestone #2 MVP 2.0 (Flipper 2.0)
Advanced version of Solidity Flipper contract, which uses local variables, error handling, control structures and operators is compatible with Nysa.

## Milestone #3 Inherited Flipper
Solidity Flipper contract now implements an Interface and inherits an abstract Flipper contract.
Additionally, Pragma section is read by transpiler. Contracts written in Solidity below 0.8.0 are rejected, 0.8.* are compiled.

## Milestone #4 ERC20
ERC20 contract is compatible with Nysa. This milestone requires working events. Additionally, we wish to implement support for imports and external contract calls.

## Milestone #5 Structs and scoping
This milestone focuses on more advanced solidity features, such as Special Variables and Functions, conversions between elementary types. At this point conversion of structs and enums will be available.
Solidity contracts which uses those features are compatible with Nysa

## Milestone #6 Function modifiers
This milestone focuses on some unique Solidity features like function modifiers, time units, ether units, and immutable state variables.

## Milestone #7 Libraries
This milestone aims to cover the missing solidity features like libraries, using for and other advanced langauge structures. Secondary goal is to fully cover with tests order of evaluation of expressions.

## Milestone #8 OpenZeppelin
Solidity contracts written using OpenZeppelin framework are compatible with Nysa.

## Milestone #8 Docs
Advanced version of Solidity Flipper contract, which uses local variables, error handling, control structures and operators is compatible with Nysa.

## Milestone #9 Solidity tests
Tests written in solidity can be transpiled into rust tests.

## Milestone #10 JS E2E tests
E2E tests written for OpenZeppelin can be run against NEAR Virtual Machine. The tests are passing.