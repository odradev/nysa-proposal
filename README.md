# Nysa Proposal
The goal of the Nysa project is to translate contracts written in Solidity to Rust. With this approach, smart contracts will be as efficient as those written using the native framework. Our initial tests are yielding very promising results.
TODO!(put some numbers)

## What problem do we solve?
The main advantage of smart contracts written in Solidity is that they are already
written and battle-tested - hence considered secure.
Platforms that offer writing smart contracts in Rust suffer from small amount
of libraries and example code, so they look at Solidity codebase with jealousy.

## What do we offer?
We strongly believe there is a better solution: **transpiling Solidity code into Rust** code.
It allows us to take advantage of the already existing Rust to WASM compilation pipeline.
In return, we get the same neat, easy, and secure but native code.

## How we do it?
In terms of programming languages, the list of Solidity features can be treated
as a subset of Rust's features.
That means it's possible to convert Solidity into Rust, but the opposite is not possible.
Our architecture is split into 3 main pieces:

- `Solidity Parser` build on top of [lalrpop](https://github.com/lalrpop/lalrpop).
  It parses Solidity Syntax into developer-friendly Solidity AST.
  We have borrowed it as a whole from [the Solang project](https://github.com/hyperledger/solang).
  Further development might require some modification, but it can be considered 95% complete.

- `C3 Lang` is our approach to implement Solidity's (and Python's) inheritance model.
  We designed it as an intermediate target for the transpiler.
  While it works, it needs better tests, redesign, and documentation.

- `Nysa Core` translates the output of the `Solidity Parser` into a `C3 Lang` Rust.

The pipeline looks like this:
```
Solidity Code -> Solidity Parser -> Solidity AST -> C3 Lang AST -> Rust Code
```

## Nysa codebase
We managed to implement the bare minimum to transpile a simple Solidity code.
It lives in [the Nysa repository](https://github.com/odradev/nysa).

## Owned Token example
We have a non-trivial example implemented - OwnedToken with is a basic Erc20 token with
ownership.

Solidity code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;

    event Transfer(address indexed from, address indexed to, uint256 value);

    constructor(string memory _name, string memory _symbol, uint8 _decimals, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _initialSupply * 10**uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
    }

    function _transfer(address _from, address _to, uint256 _value) internal {
        require(_to != address(0), "Invalid recipient address.");
        require(balanceOf[_from] >= _value, "Insufficient balance.");

        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;

        emit Transfer(_from, _to, _value);
    }

    function transfer(address _to, uint256 _value) public {
        _transfer(msg.sender, _to, _value);
    }
}
```

It is transpiled into the following Rust code:
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

### Odra vs Nysa
Compare

# Timeline
Proposed list of milestones.

## Milestone #1 MVP (UniswapV2)
This milestone aims to transpile Uniswap v2 [core contracts](https://github.com/Uniswap/v2-core/blob/master/contracts).
The code can be transpiled to Rust (compatible with the Odra Framework) and compiled into a wasm file.

Translating Uniswap v2 contracts from Solidity to Rust requires handling all the fundamental language features, starting from contract declarations, interfaces, state definition, function invocations, event emission, error handling, ensuring type compatibility, all the way to calling external contracts.

Additionally, besides purely syntactic issues, addressing the inheritance model in Solidity, contextual parsing of code, and creating contract factory are also necessary.

Covered Solidity features:
- Functions
    * modifiers,
    * constructors,
    * function calls,
    * returns (includes returing named values).
- Contract state variables 
    * single values,
    * mappings,
    * nested mappings,
    * assiging default value,
    * default getters for the public variables.
- Types
    * integers (singned/unsigned),
    * bool,
    * string,
    * address,
    * arrays,
    * bytes,
    * custom enums and structs
- Checked/unchecked arithmetic.
- Local variables.
- Error handling.
- Control Structures
    * if/else
    * while loops
    * tenary operator ( ? :).
- Operators.
    * logical operators,
    * math operators,
    * bitwise operators.
- Basic special variables and functions
    * msg.sender
    * msg.value
    * block.timestamp
    * abi.encode
    * abi.decode
    - Inheritance.
- Events.
- External contract calls.
- Libraries
- [Salted contract creations / create2](https://docs.soliditylang.org/en/v0.8.21/control-structures.html#salted-contract-creations-create2)

## Milestone #3 Full syntax support

Although the functionality set required by Uniswap v2 smart contracts is relatively rich, it does not exhaust all that is available in Solidity. The second milestone aims to address the missing language features in terms of syntax as well as idiomatic expressions.

The secondary goal is to fully cover with tests the order of evaluation of expressions.

Solidity contracts written using the OpenZeppelin framework are compatible with Nysa.

## Milestone #4 Docs
The documentation of all the supported features and discrepancies to EVM are documented.
It includes tutorials and an introduction for Solidity developers.

## Milestone #5 Solidity tests
Tests written in solidity can be transpiled into Rust tests.

E2E JavaScript tests written for OpenZeppelin can be run against the Casper Virtual Machine provided with the Odra Framework.
The tests are passing.
